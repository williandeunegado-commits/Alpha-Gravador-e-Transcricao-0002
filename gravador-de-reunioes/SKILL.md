---
name: gravador-de-reunioes
description: Instala (ou atualiza) um módulo de gravação e transcrição de reuniões dentro de um sistema já existente — grava áudio no navegador, transcreve com IA em blocos, extrai um resumo e itens de ação combinados (cada um com código de identificação, responsável, prazo e um campo de observação livre), oferece um card para já deixar a próxima 1:1 agendada logo após a revisão, e exige revisão humana antes de qualquer combinado virar tarefa de verdade. Use sempre que o usuário pedir para instalar/criar um "gravador de reuniões", "gravador de 1:1", transcrição automática de reunião, ata de reunião com IA, resumo automático de call, ou enviar este arquivo pedindo para instalar esse módulo no projeto atual — mesmo que ele descreva a ideia com outras palavras (gravar uma conversa, transcrever e listar o que ficou combinado, separar quem falou o quê).
---

# Gravador de Reuniões — instalador

## O que esta skill faz

Esta skill não cria um app do zero — ela adiciona uma funcionalidade completa de
gravação/transcrição/resumo de reuniões dentro de um sistema que a pessoa já tem
rodando. Isso muda como você deve agir: antes de escrever qualquer arquivo, você
precisa entender o projeto onde vai mexer (como ele está estruturado, qual banco
usa, como ele guarda segredos), porque não existe um único jeito "certo" de
implementar isso — só um conjunto de decisões de arquitetura que já foram
testadas em produção e que valem a pena preservar.

O pipeline, em uma frase: **grava no navegador → sobe o áudio → corta em blocos
→ transcreve cada bloco com IA → junta tudo → um segundo modelo extrai resumo e
itens de ação sugeridos → um humano confirma os itens antes deles virarem tarefa
→ o áudio é descartado**. Cada uma dessas setas existe por um motivo específico
explicado abaixo — não pule etapas achando que está simplificando.

## Antes de escrever código: entreviste a pessoa

Não assuma nada sobre o projeto onde você está instalando isso — pergunte. Faça
isso de forma natural, numa conversa, não como um formulário. Os pontos que
realmente mudam o resultado:

1. **Nome do módulo.** O padrão é "Gravador de Reuniões", mas pergunte se a
   pessoa quer outro nome — e use esse nome de forma consistente (nome de
   pasta, nome de tabela, textos de interface). Não deixe "v2" nem nomes
   genéricos tipo "meeting-module" se ela já deu um nome.
2. **Stack do projeto.** Pergunte com que framework/backend e banco de dados o
   projeto já trabalha (ex.: Next.js + Supabase, Rails + Postgres, Django,
   um backend próprio em Node). Isso decide qual referência você vai seguir:
   - Next.js + Supabase (ou muito parecido) → leia `references/stack-nextjs-supabase.md`.
   - Qualquer outra coisa → leia `references/stack-generic.md` e adapte os
     nomes de tecnologia para o equivalente do projeto.
   Se a pessoa não souber responder com precisão, explore o repositório você
   mesmo (leia `package.json`, arquivos de config, pasta de migrations) antes
   de perguntar de novo — só volte a perguntar o que não conseguiu inferir.
3. **Diarização (saber quem falou o quê).** Explique antes de perguntar: em
   produção, pedir para o modelo identificar quem estava falando fez a
   transcrição de áudios longos "degenerar" (o modelo travava repetindo a
   mesma frase tentando adivinhar o interlocutor). Sem diarização, a
   transcrição fica só com timestamp, é mais confiável, e ainda cobre bastante
   coisa (a pessoa pode ver quem estava na reunião pelo campo de
   participantes). Pergunte se ela quer: (a) instalar sem diarização
   (recomendado), ou (b) instalar com diarização, aceitando o risco e usando
   as redes de segurança reforçadas descritas em
   `references/prompts-e-seguranca.md`. Registre a escolha no código (uma
   constante ou variável de ambiente), não deixe isso "hardcoded" demais para
   não dar para desligar depois se der problema.
4. **Retenção de áudio.** O padrão desta instalação é **nunca guardar o áudio
   de forma permanente** — ele existe só durante o processamento e é apagado
   assim que a transcrição termina (com sucesso ou com erro). Confirme isso
   com a pessoa e deixe claro o trade-off: sem o áudio guardado, se a
   transcrição falhar o único jeito de recuperar é gravar de novo. Se ela
   preferir manter o áudio por um tempo (ex.: até a transcrição ser aprovada),
   ajuste — mas não guarde por padrão sem perguntar, já que isso tem
   implicação de privacidade.
5. **Participantes e permissões.** Pergunte quem pode gravar/ver a transcrição
   de uma reunião no sistema dela (ex.: só quem participou? um gestor e a
   pessoa? qualquer um da equipe?). Isso define a regra de permissão que você
   vai implementar — não invente uma regra genérica sem confirmar.
6. **Chaves de API disponíveis.** A transcrição usa a Gemini Files API
   (`GEMINI_API_KEY`) e a extração/estruturação usa a API da Anthropic
   (`ANTHROPIC_API_KEY` ou o SDK que o projeto já usa para chamar Claude). Se
   alguma dessas variáveis não existir no projeto, diga exatamente o nome que
   precisa ser adicionado ao `.env` — nunca invente uma chave de exemplo nem
   peça para a pessoa colar a chave no chat.

## Verificações antes de instalar

- **Repositório git**: rode algo como `git rev-parse --is-inside-work-tree`
  no diretório do projeto. Se não houver repositório, avise a pessoa e
  pergunte se quer que você inicialize um antes de continuar — não
  inicialize silenciosamente.
- **`ffmpeg` e `ffprobe`**: são necessários no ambiente que vai rodar a
  transcrição, porque o áudio gravado no navegador não tem cabeçalho de
  duração e a Gemini Files API rejeita isso — o áudio precisa ser
  recodificado e cortado em blocos antes. Verifique se já estão disponíveis
  no ambiente de deploy do projeto (ex.: Dockerfile, buildpack) e avise se
  precisar adicionar.
- **Onde o código roda**: confirme se o processamento pode rodar dentro do
  ciclo de vida normal de uma requisição HTTP (com timeout maior) ou se o
  projeto já tem fila/worker em background — isso muda onde o código de
  transcrição deve morar.

## Passo a passo da instalação

Depois de ter as respostas acima, siga esta ordem — cada etapa depende da
anterior, então não pule para escrever código antes de confirmar o essencial:

1. Confirme com a pessoa um resumo curto do que você vai construir (nome do
   módulo, stack, diarização sim/não, política de áudio) antes de criar
   qualquer arquivo. É mais barato corrigir um mal-entendido aqui do que
   depois de gerar dez arquivos.
2. Leia o arquivo de referência da stack escolhida
   (`references/stack-nextjs-supabase.md` ou `references/stack-generic.md`)
   e o arquivo de prompts/segurança (`references/prompts-e-seguranca.md`).
3. Crie o modelo de dados primeiro (migration/schema): tabela principal da
   reunião, tabela de itens de ação (que atravessa reuniões, não é recriada
   a cada uma) e uma tabela de log de edição da transcrição (apenas
   inserção, nunca update/delete — é o que permite corrigir "alucinações"
   sem perder o histórico). Ver a seção de modelo de dados na referência de
   stack.
4. Implemente a gravação no navegador (componente/hook) com as três
   proteções descritas na seção "Arquitetura de referência" abaixo (aba
   fechada, atualização de página, microfone negado/gravação vazia).
5. Implemente o upload (rota separada, nunca o navegador falando direto com
   o storage) e a rota de processamento (separada do upload, porque
   transcrever demora e subir arquivo é rápido — juntar as duas etapas numa
   só arrisca estourar timeout no meio do upload).
6. Implemente a transcrição em blocos com escalada de modelo e as redes de
   segurança contra degeneração (repetição de linha/frase, cobertura de
   timestamp, `finishReason` inesperado) — está tudo detalhado em
   `references/prompts-e-seguranca.md`. Garanta que o áudio é apagado do
   storage temporário (e do storage da Gemini) depois, mesmo se der erro.
7. Implemente a extração (segunda chamada de IA, agora para estruturar
   resumo + itens sugeridos em JSON) com parsing defensivo — nunca deixe uma
   resposta malformada derrubar o pipeline inteiro; trate isso como erro
   explícito e devolva algo vazio em vez de quebrar.
8. Implemente a tela de revisão dos itens sugeridos: cada item precisa de um
   **código curto de identificação** (não é a chave interna do banco — é um
   número sequencial exibido para a pessoa referenciar o combinado numa
   conversa, ex. "Combinado #12"), de um **seletor de responsável** (a
   partir das pessoas reais do sistema, não do texto livre que a IA
   sugeriu), de um **campo de prazo** (data), e de um **campo de observação
   livre** — separado da descrição do combinado, para anotar contexto extra
   que a pessoa queira registrar na revisão (não é preenchido pela IA). Só
   depois que a pessoa confirma é que o item vira uma linha de verdade na
   tabela — a IA nunca grava um combinado sozinha.
9. Depois que os itens forem confirmados, mostre um **card perguntando se a
   pessoa já quer deixar a próxima reunião 1:1 agendada** (seletor de
   data/hora). Se ela confirmar, crie automaticamente a próxima reunião com
   status "agendada" e vinculada a esta (não abra gravação nenhuma, só a
   agenda). Se ela recusar ou pular o card, não crie nada — ela sempre pode
   agendar manualmente depois.
10. Implemente a correção de trechos da transcrição (edição inline) que
    grava no log apenso, sem nunca sobrescrever o histórico.
11. Aplique a regra de permissão que a pessoa confirmou no passo 5 da
    entrevista, centralizada num único lugar (função/policy reaproveitada),
    não repetida em cada rota.
12. Rode a checagem de build/lint/typecheck que o projeto já usa, se
    houver, para pegar erros óbvios antes de entregar.
13. Resuma para a pessoa: o que foi criado, quais variáveis de ambiente
    ainda precisam ser configuradas, e o que falta fazer manualmente (ex.:
    rodar a migration, testar uma gravação curta).

## Arquitetura de referência (o porquê de cada peça)

Isso vem de uma implementação real em produção — cada decisão aqui evita um
problema que já aconteceu, então generalize os detalhes técnicos para a
stack do projeto, mas não jogue fora a lógica por trás:

- **Três guardas na gravação**: descartar o blob se a aba fechar no meio
  (só uma parada explícita deve virar reunião registrada), pedir
  confirmação antes de sair da página enquanto grava, e não iniciar o
  upload se a gravação estiver vazia. Sem isso, é fácil acumular "reuniões"
  fantasmas no banco.
- **Upload e processamento em chamadas separadas**: subir o arquivo é
  rápido, transcrever não é — combinar as duas coisas numa só requisição
  arrisca timeout no meio do upload, que é o pior momento para falhar.
- **Cortar o áudio em blocos antes de transcrever**: contêineres de áudio
  gravados via `MediaRecorder` costumam não ter cabeçalho de duração, e
  isso quebra a maioria das APIs de transcrição por arquivo. Recodificar
  uma vez e cortar sem reencodar de novo resolve isso e ainda permite
  paralelismo controlado.
- **Escalada de modelo (barato → caro)**: tentar primeiro com o modelo mais
  barato e só escalar para um mais caro se falhar equilibra custo e taxa de
  sucesso — é, na prática, um controle de custo por si só.
- **Separar "transcrever" de "extrair estrutura" em duas chamadas de IA**:
  cada uma fica mais simples de acertar e mais barata de depurar do que uma
  chamada tentando fazer as duas coisas.
- **Nunca deixar a IA gravar um combinado sozinha**: ela sugere texto; uma
  pessoa escolhe o responsável de verdade e confirma antes de qualquer
  linha virar tarefa. Nome errado ou responsável errado custa caro o
  suficiente para justificar essa etapa manual.
- **Log de edição apenso, nunca sobrescrito**: cada correção de
  "alucinação" da transcrição vira uma linha nova de histórico. Isso também
  significa que corrigir um erro não deveria exigir reabrir a reunião como
  se ela ainda estivesse em andamento.
- **Áudio nunca reproduzível e, nesta instalação, nunca retido**: ninguém
  ouve o áudio de volta — só a transcrição é consultável, e o áudio em si
  é descartado depois do processamento (ver a decisão de retenção na
  entrevista).
- **Card de agendar a próxima 1:1 é uma sugestão, não uma ação automática
  silenciosa**: ele só cria a próxima reunião se a pessoa escolher uma
  data/hora e confirmar — nunca agenda sozinho no fundo, para não conflitar
  com a agenda real de quem organiza recorrência de 1:1s.

## Onde estão os detalhes técnicos

Carregue só o que for relevante para a instalação atual:

- `references/stack-nextjs-supabase.md` — layout de arquivos, rotas, schema
  SQL e função de permissão para projetos Next.js + Supabase (ou muito
  parecidos).
- `references/stack-generic.md` — os mesmos componentes traduzidos para
  qualquer outra stack, com o que adaptar em cada camada.
- `references/prompts-e-seguranca.md` — os prompts de transcrição (com e
  sem diarização), o prompt de extração, as redes de segurança contra
  degeneração do modelo, e as regras de parsing defensivo.

## Princípio de não-surpresa

Nunca exponha uma chave de serviço (banco, storage) no código que roda no
navegador — toda escrita sensível passa por uma rota de servidor. Nunca deixe
um processo salvar dados de forma que surpreenda a pessoa depois (ex.: manter
áudio guardado quando ela pediu para não guardar, ou deixar a IA gravar um
item de ação sem revisão). Se em algum ponto a pessoa pedir para pular uma
dessas proteções "só para simplificar", explique o motivo da proteção antes
de topar — a explicação geralmente já resolve, porque a pessoa não tinha o
contexto do problema que aquilo evita.

## Depois de instalar

Sugira um teste curto e de baixo risco antes de considerar a instalação
concluída: gravar 30-60 segundos falando algo com um combinado claro (ex.:
"Fulano vai enviar o relatório até sexta"), rodar o pipeline inteiro, e
conferir se o resumo, o item de ação sugerido e a correção manual de um
trecho funcionam de ponta a ponta. Isso pega problemas de configuração
(chave de API errada, variável de ambiente faltando, permissão de
storage) muito mais rápido do que revisar o código estaticamente.
