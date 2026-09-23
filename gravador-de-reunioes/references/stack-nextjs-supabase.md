# Stack: Next.js + Supabase

Use esta referência quando o projeto já usa Next.js (App Router ou Pages) e
Supabase (Postgres + Storage + Auth). Os nomes de arquivo/tabela abaixo são
sugestões — ajuste ao padrão de nomenclatura que o projeto já usa, mas
mantenha a separação de responsabilidades.

## Layout de arquivos sugerido

```
app/api/<modulo>/[reuniaoId]/audio/route.ts       ← upload do áudio
app/api/<modulo>/[reuniaoId]/processar/route.ts   ← dispara transcrição + extração
app/api/<modulo>/[reuniaoId]/transcricao/route.ts ← PATCH para corrigir um trecho
app/api/<modulo>/[reuniaoId]/itens/route.ts        ← POST para confirmar itens revisados
app/api/<modulo>/[reuniaoId]/proxima/route.ts      ← POST para agendar a próxima 1:1 (card pós-revisão)
components/<Modulo>/Gravador.tsx                   ← wrapper do MediaRecorder
lib/<modulo>/transcricao.ts                        ← chunking + chamadas Gemini
lib/<modulo>/extracao.ts                           ← chamada Claude + parsing defensivo
lib/<modulo>/permissoes.ts                          ← wrapper da função SQL de permissão
supabase/migrations/<timestamp>_<modulo>.sql        ← schema completo
```

Troque `<modulo>` e `<Modulo>` pelo nome que a pessoa escolheu na entrevista.

## Gravação (componente)

- `navigator.mediaDevices.getUserMedia({ audio: true })` dentro de um
  try/catch simples; em caso de erro, mostrar um aviso pedindo para checar a
  permissão do microfone — não precisa de máquina de estados própria para
  isso.
- `new MediaRecorder(stream, { mimeType: 'audio/webm;codecs=opus' })`, com
  fallback para `audio/webm` puro se o navegador não suportar o codec.
- `recorder.start(10_000)` para emitir pedaços a cada 10s — isso é só uma
  salvaguarda contra perder tudo numa gravação longa; os pedaços só viram um
  `Blob` único quando `stop()` é chamado.
- No `onstop`, cheque `blob.size === 0` antes de subir; se vazio, avise e não
  faça upload.
- Um listener de `beforeunload` chamando `preventDefault()` enquanto o estado
  não for "parado".
- Uma `ref` marcada no `unmount` para descartar o blob em vez de fazer
  upload se a aba fechar no meio da gravação.

## Upload (`.../audio/route.ts`)

- Recebe `FormData` com o blob, grava no bucket do Supabase Storage usando a
  **service role key** (nunca a chave anônima) — o bucket não deve ter
  nenhuma policy que permita escrita direta do usuário final.
- Caminho sugerido: `{organization_id}/{reuniao_id}/audio.webm`, com
  `upsert: true` (regravar substitui, nunca acumula versão).
- Cheque o tamanho pelo header `Content-Length` **antes** de ler o corpo
  inteiro — não espere `await req.formData()` terminar para descobrir que o
  arquivo é grande demais.
- Se o registro no banco falhar depois do upload, apague o arquivo que
  acabou de subir — nunca deixe um objeto órfão no bucket.
- Como a política de retenção desta instalação é não guardar áudio de forma
  permanente (ver entrevista no SKILL.md principal), marque este arquivo
  como temporário: apague-o do bucket assim que a rota de processamento
  terminar (com sucesso ou erro), em vez de deixá-lo lá indefinidamente.

## Transcrição em blocos (`lib/<modulo>/transcricao.ts`)

1. Recodifique o `.webm` para `ogg/opus` mono com `ffmpeg` (o contêiner do
   `MediaRecorder` não tem cabeçalho de duração e a Gemini Files API rejeita
   isso direto) e corte em blocos de 900s (15min) com
   `-f segment -c copy` (sem reencodar de novo). Descarte um resto menor que
   ~15s.
   - Se a diarização estiver ligada, reduza o tamanho do bloco (ex.: 480s /
     8min) — blocos menores dão ao modelo menos chance de "perder o fio" de
     quem está falando. Ver `references/prompts-e-seguranca.md`.
2. Por bloco: upload resumível para a Gemini Files API
   (`X-Goog-Upload-Protocol: resumable`), poll de status a cada 3s (até ~3min,
   tolerando algumas falhas 429/5xx seguidas), depois transcrição via
   streaming (`streamGenerateContent?alt=sse`) com um watchdog que aborta se
   ficar mais de 120s sem receber nada.
3. Rode 2 blocos em paralelo (worker pool simples com `Promise.all`) — o
   suficiente para reduzir o tempo total sem estourar limite de taxa.
4. Escada de modelo: tente o modelo mais barato duas vezes antes de escalar
   para um modelo mais caro/capaz. Se um único bloco falhar depois de
   esgotar toda a escada, trate a transcrição inteira como falha — uma
   transcrição parcial não é confiável o suficiente para seguir adiante.
5. Apague o arquivo temporário da Gemini depois de transcrever
   (best-effort — não precisa bloquear o fluxo se isso falhar).
6. Depois de transcritos, recombine os blocos ajustando os timestamps pelo
   deslocamento acumulado (cada bloco pensa que começa em `0:00`).

Isso tudo roda dentro do ciclo de vida normal da requisição (aumente o
timeout do servidor, ex. `maxDuration` no Next.js) se o projeto não tiver
fila/worker em background — confirme isso na entrevista.

## Extração (`lib/<modulo>/extracao.ts`)

Chamada ao Claude na mesma requisição, logo depois da transcrição terminar.
Ver o prompt completo em `references/prompts-e-seguranca.md`. Pontos-chave de
implementação:

- Limpe a resposta de cercas ```` ```json ```` antes de fazer o parse; se o
  parse falhar, devolva um objeto vazio (`{ resumo: '', itens: [] }`) em vez
  de derrubar o pipeline inteiro.
- Se `stop_reason` indicar que a resposta foi cortada por limite de tokens,
  trate como erro explícito — não devolva um JSON truncado.
- O resumo pode ser salvo direto no banco (baixo risco). Os itens sugeridos
  **não** — eles voltam só na resposta da rota, para a pessoa revisar antes
  de confirmar.

## Próxima reunião (`.../proxima/route.ts`)

Depois que a pessoa confirma os itens revisados (POST `.../itens/route.ts`), a
tela mostra um card perguntando se ela já quer deixar a próxima 1:1 agendada,
com um seletor de data/hora.

- Se ela escolher uma data e confirmar, esta rota faz **apenas um insert** em
  `<modulo>_reunioes`: `status = 'agendada'`, `data_hora_agendada` com o valor
  escolhido, `reuniao_anterior_id` apontando para a reunião que acabou de ser
  revisada, `organization_id` e `criado_por` herdados da reunião anterior.
- Se ela pular ou fechar o card sem escolher data, não chame esta rota — não
  crie uma reunião "agendada" sem data nem hora.
- Esta rota nunca dispara e-mail/notificação por conta própria nem grava em
  agenda externa (Google Calendar etc.) a menos que o projeto já tenha essa
  integração e a pessoa peça explicitamente — o padrão é só criar o registro
  interno.

## Modelo de dados (SQL)

```sql
create table <modulo>_reunioes (
  id uuid primary key default gen_random_uuid(),
  organization_id uuid not null references organizations(id),
  -- ajuste os papéis abaixo ao seu domínio (gestor/colaborador, anfitrião/convidados, etc.)
  criado_por uuid not null references profiles(id),
  status text not null default 'agendada', -- agendada | realizada | cancelada
  data_hora_agendada timestamptz,
  reuniao_anterior_id uuid references <modulo>_reunioes(id), -- preenchido quando esta reunião nasceu do card "agendar a próxima 1:1"
  transcricao text,
  transcricao_status text not null default 'nao_iniciada', -- nao_iniciada | processando | pronta | erro
  resumo text,
  observacoes text,
  created_at timestamptz not null default now(),
  constraint conteudo_minimo check (status = 'agendada' or observacoes is not null or resumo is not null)
);

create table <modulo>_itens (
  id uuid primary key default gen_random_uuid(),
  codigo bigint generated always as identity, -- número curto e sequencial só para exibir e referenciar ("Combinado #12"); nunca use como chave de junção
  reuniao_id uuid not null references <modulo>_reunioes(id),
  descricao text not null check (char_length(descricao) between 1 and 500),
  observacao text, -- nota livre da pessoa na revisão, separada da descrição; nunca preenchida pela IA
  responsavel_id uuid references profiles(id),
  prazo date,
  status text not null default 'aberto', -- aberto | feito | abandonado
  motivo text, -- obrigatório quando status = 'abandonado'; valide na aplicação
  fechado_em timestamptz,
  fechado_por uuid references profiles(id)
);

create table <modulo>_transcricao_edicoes (
  id uuid primary key default gen_random_uuid(),
  reuniao_id uuid not null references <modulo>_reunioes(id),
  indice_trecho int not null,
  timestamp_trecho text,
  texto_anterior text,
  texto_novo text,
  editado_por uuid not null references profiles(id),
  created_at timestamptz not null default now()
);
-- sem policy de UPDATE/DELETE nesta tabela: é um log apenso de verdade,
-- garantido pelo RLS, não só por convenção de código.
```

A transcrição fica inteira numa única coluna de texto — não crie uma tabela
de segmentos. Divida por quebra de linha em código toda vez que a tela é
aberta (o índice é só a posição no array; guarde também o
`timestamp_trecho` original em cada edição, para conferência manual caso o
índice desalinhe depois de uma edição que junte/separe linhas).

## Permissões (RLS)

Centralize a regra numa única função SQL `SECURITY DEFINER` (evita
recursão de RLS lendo a própria tabela de dentro da policy) e reaproveite em
toda tabela do módulo, em vez de reescrever a mesma regra em cada uma. Use a
resposta da entrevista (quem pode gravar/ver) para definir as condições —
não copie a regra do exemplo (gestor/colaborador) se o domínio da pessoa for
diferente.

Padrão útil de separar leitura de escrita: quem pode *ver* geralmente é um
grupo maior que quem pode *escrever* (ex.: qualquer participante vê a
transcrição, mas só quem organizou ou tem um papel administrativo pode
corrigir um trecho ou apagar um item).
