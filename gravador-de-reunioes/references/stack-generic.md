# Stack: qualquer outra (fora Next.js + Supabase)

Use esta referência quando o projeto usar outro framework, banco de dados ou
storage. A arquitetura em si não depende de nenhum fornecedor específico — o
que muda é só onde cada peça de infraestrutura entra. Traduza cada item
abaixo para o equivalente real do projeto antes de escrever código; não deixe
nomes de tecnologia genéricos ("seu banco", "seu storage") no código final.

## Componentes que você precisa criar, independente da stack

| Peça | Papel | Equivalentes comuns |
|---|---|---|
| Gravação no navegador | Captura o áudio com `MediaRecorder` (API nativa do browser, não muda com a stack do backend) | — |
| Rota de upload | Recebe o áudio, grava num storage que o navegador não acessa diretamente | S3, GCS, Azure Blob, disco local (dev), bucket privado |
| Rota de processamento | Corta o áudio, chama a API de transcrição, chama a API de extração | rota HTTP comum, ou um job de fila se o projeto já tiver worker |
| Fila/worker (opcional) | Roda o processamento fora do ciclo de vida da requisição HTTP, se o projeto já tiver essa infraestrutura | Sidekiq, BullMQ, Celery, etc. |
| Banco de dados | Guarda reunião, itens de ação e log de edição | Postgres, MySQL, Mongo — o desenho relacional abaixo se adapta com pequenos ajustes a NoSQL |
| Permissão | Decide quem pode gravar, ver e corrigir | função/policy do banco, middleware da aplicação, camada de autorização própria |

O design (chave de serviço, sem acesso direto do navegador ao storage, path
por organização/entidade) se mantém igual não importa qual storage você
escolher — só troque o SDK.

## Gravação no navegador

Igual em qualquer stack, porque é API do browser, não do backend:

- `getUserMedia({ audio: true })` num try/catch simples.
- `MediaRecorder` com `audio/webm;codecs=opus` (fallback para `audio/webm`
  puro).
- `start(10_000)` para pedaços de 10s como salvaguarda; só vira um blob
  único no `stop()`.
- Três guardas: descartar se a aba fechar no meio, confirmar antes de sair
  da página enquanto grava, e não subir se o blob estiver vazio
  (`size === 0`).

## Upload e processamento

- Upload e processamento são **duas chamadas separadas**, nunca uma só —
  subir o arquivo é rápido, transcrever não é.
- A rota de upload grava com uma credencial de servidor (nunca uma chave que
  o navegador possa usar para escrever direto no storage).
- Cheque o tamanho pelo cabeçalho da requisição antes de carregar o corpo
  inteiro na memória, não depois.
- Como a política de retenção desta instalação é não guardar áudio de forma
  permanente, apague o arquivo do storage assim que o processamento
  terminar — com sucesso ou com erro.

## Transcrição em blocos

Independente do provedor de transcrição (esta instalação usa a Gemini Files
API, ver `references/prompts-e-seguranca.md`):

1. Recodifique o áudio com `ffmpeg` para um formato com cabeçalho de duração
   normal (a maioria das APIs de transcrição por arquivo rejeita o
   contêiner de streaming que o `MediaRecorder` gera) e corte em blocos de
   ~15 minutos sem reencodar de novo. Se diarização estiver ligada, use
   blocos menores (~8min).
2. Processe os blocos com paralelismo controlado (2-3 ao mesmo tempo é
   suficiente na maioria dos casos) e uma escada de modelo barato → caro.
3. Depois de transcritos, junte os blocos ajustando os timestamps pelo
   deslocamento acumulado.
4. Se não houver fila/worker no projeto, rode tudo dentro do ciclo de vida
   da requisição HTTP com um timeout de servidor maior. Se houver, prefira
   mover o processamento para um job em background e deixar o front-end
   fazer polling do status.

## Extração

Uma segunda chamada de IA (Claude), separada da transcrição, que recebe o
texto e devolve JSON estruturado (resumo + itens sugeridos). Faça parsing
defensivo: nunca deixe uma resposta malformada derrubar o pipeline — trate
como erro explícito e devolva algo vazio.

## Modelo de dados

Três entidades, com o mesmo desenho relacional independente do banco
escolhido:

1. **Reunião** — dados da reunião em si: status, transcrição (texto único,
   sem tabela de segmentos), resumo, observações, e uma referência opcional
   à reunião anterior (`reuniao_anterior_id` ou equivalente), preenchida
   quando ela nasceu do card "agendar a próxima 1:1".
2. **Itens de ação** — entidade própria, referenciando a reunião, porque
   eles atravessam reuniões (não são recriados a cada uma). Campos: um
   código curto e sequencial só para exibição/referência (ex. "Combinado
   #12" — não é a chave interna), descrição, observação livre (nota da
   pessoa na revisão, separada da descrição, nunca preenchida pela IA),
   responsável, prazo, status, motivo (quando abandonado).
3. **Log de edição da transcrição** — tabela apenas de inserção (sem
   update/delete), uma linha por correção: índice do trecho, timestamp
   original, texto anterior, texto novo, quem editou. Em bancos sem suporte
   nativo a "somente inserção", aplique essa regra na camada de aplicação e
   documente que nunca deve ser violada.

Se o banco for NoSQL, mantenha a mesma separação em coleções/documentos
distintos em vez de aninhar tudo dentro do documento da reunião — isso evita
reescrever o documento inteiro a cada correção de trecho ou a cada mudança
de status de um item.

## Próxima reunião (agendamento no fim da revisão)

Depois que a pessoa confirma os itens revisados, mostre um card perguntando
se ela já quer deixar a próxima 1:1 agendada, com um seletor de data/hora:

- Se ela escolher uma data e confirmar, crie **apenas um registro** de
  reunião com status "agendada", a data/hora escolhida, e a referência à
  reunião anterior — sem abrir gravação nenhuma.
- Se ela pular ou fechar o card sem escolher data, não crie nada — nunca
  registre uma reunião "agendada" sem data nem hora.
- Não dispare e-mail/notificação nem grave em agenda externa por conta
  própria, a menos que o projeto já tenha essa integração e a pessoa peça
  explicitamente — o padrão é só o registro interno.

## Permissões

Centralize a regra de "quem pode ver" e "quem pode escrever" num único lugar
reaproveitado por toda a funcionalidade (uma função, um middleware, uma
camada de autorização), em vez de reimplementar a mesma checagem em cada
rota. Use a resposta da entrevista sobre quem pode gravar/ver — não copie um
modelo de papéis (gestor/colaborador) que não corresponda ao domínio da
pessoa.
