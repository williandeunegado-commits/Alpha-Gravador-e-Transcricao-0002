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
