# Gravador de Reuniões — skill para instalar num sistema já existente

## O que isso faz

Essa é uma **skill do Claude**: um arquivo de instruções que ensina o Claude
a fazer uma tarefa específica de forma consistente. Essa aqui ensina o
Claude a adicionar, dentro de um sistema que você já tem, uma funcionalidade
completa de **gravar reuniões, transcrever com IA, resumir e listar o que
ficou combinado** — sem que você precise descrever a arquitetura toda de
novo cada vez que quiser instalar isso num projeto.

Depois de instalada, quando você pedir para o Claude "instalar o gravador de
reuniões" (ou algo parecido) dentro de um projeto, ele vai te fazer algumas
perguntas rápidas (nome do módulo, qual stack você usa, se quer identificar
quem falou o quê) e então escrever o código necessário diretamente no seu
projeto.

Cada item de ação (combinado) vem com um código curto de identificação, um
seletor de responsável, um campo de prazo e um campo de observação livre
para anotar contexto extra. Depois que os itens são revisados e confirmados,
a tela ainda mostra um card perguntando se você já quer deixar a próxima
reunião 1:1 agendada.

## O que tem dentro do módulo

**Gravação e transcrição**
- Gravação de áudio direto no navegador, com proteções contra aba fechada, atualização de página e gravação vazia.
- Upload do áudio por rota de servidor (o navegador nunca escreve direto no storage).
- Transcrição com IA (Gemini) em blocos, com escalada de modelo (barato para caro) e redes de segurança contra travamento/repetição.
- Diarização (quem falou o quê) opcional, desligada por padrão.
- Áudio descartado assim que o processamento termina (nunca guardado de forma permanente).

**Resumo e combinados**
- Resumo automático da reunião (Claude), separado da transcrição.
- Itens de ação sugeridos pela IA, sempre revisados por uma pessoa antes de virarem tarefa. Cada item tem:
  - código curto de identificação (ex.: "Combinado #12");
  - descrição do que ficou combinado;
  - responsável (escolhido entre as pessoas reais do sistema);
  - prazo (data);
  - observação livre;
  - status (aberto, feito, abandonado) e motivo quando abandonado.
- Card ao final da revisão perguntando se você já quer deixar a próxima reunião 1:1 agendada.

**Transcrição e segurança**
- Correção de trechos da transcrição com histórico apenas de inserção (nunca sobrescreve).
- Regra de permissão centralizada (quem pode gravar, ver e corrigir).
- Schema SQL pronto (Next.js + Supabase) e guia equivalente para outras stacks.

**Arquivos da skill**

```
gravador-de-reunioes/
├── SKILL.md                          instruções principais (entrevista + passo a passo)
├── README.md
└── references/
    ├── stack-nextjs-supabase.md      rotas, schema SQL e permissões (Next.js + Supabase)
    ├── stack-generic.md              o mesmo, para qualquer outra stack
    └── prompts-e-seguranca.md        prompts de transcrição/extração e redes de segurança
```
## O que é uma skill? O que é o GitHub?

Uma **skill** é um arquivo de texto (`SKILL.md`) com instruções que o Claude
lê quando a tarefa pede por ele — como um manual de instruções que ele
consulta na hora certa.

O **GitHub** é como um Google Drive, só que feito para compartilhar código
— e é onde as pessoas costumam compartilhar skills como essa.

## Como instalar

Esta skill precisa ser instalada no **Claude Code** (o Claude com acesso ao
terminal e aos arquivos do seu computador) — não no site/app comum do
Claude. Isso porque a skill escreve código diretamente dentro do seu
projeto, e só o Claude Code tem acesso ao seu projeto de verdade.

Se você já usa o Claude Code, não precisa nem baixar nada:

### Opção 1 — a mais simples (colar o link e pedir)

1. Copie o link deste repositório.
2. Dentro do Claude Code (aberto na pasta do seu projeto), escreva algo
   como:
   *"Instala essa skill pra mim: `<link-deste-repositório>`"*
3. O Claude Code vai baixar o repositório sozinho e colocar a skill no
   lugar certo (`~/.claude/skills/gravador-de-reunioes/`).

### Opção 2 — um comando (mais rápido, se você já usa terminal)

```bash
npx skills add <link-deste-repositorio> --skill gravador-de-reunioes
```

### Opção 3 — manual

```bash
git clone <link-deste-repositorio>
# copie a pasta gravador-de-reunioes/ para ~/.claude/skills/gravador-de-reunioes/
```

## Como usar depois de instalada

Dentro do projeto onde você quer adicionar a funcionalidade, diga algo como:

- "Instala o gravador de reuniões nesse projeto."
- "Quero adicionar transcrição automática de reunião com resumo e lista de
  combinados aqui no sistema."
- "Cria um módulo de ata de reunião com IA, parecido com o que a gente
  discutiu."

O Claude vai te perguntar sobre a stack do seu projeto, se você quer
identificar quem falou o quê (com um aviso sobre o trade-off disso), e vai
escrever o código adaptado ao que você já tem.

## Crédito

A arquitetura usada como referência para esta skill (pipeline de gravação,
corte em blocos, escalada de modelo, separação entre transcrição e extração,
revisão humana antes de virar tarefa) foi baseada num levantamento de um
sistema de produção real de gravação de 1:1s, gentilmente documentado pelo
autor desta skill a partir do próprio código.

## Licença

_A definir — pergunte ao autor se quer adicionar uma licença (ex.: MIT) antes
de considerar este repositório pronto para publicação._
