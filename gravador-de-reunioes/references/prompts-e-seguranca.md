# Prompts e redes de segurança

## Transcrição (Gemini) — sem diarização (padrão recomendado)

```
Transcreva INTEGRALMENTE este áudio de uma reunião de trabalho
em português (ou no idioma predominante do áudio, ~15 minutos).
- Fiel e completa, sem resumir, sem omitir, sem comentar.
- A cada troca de quem fala (ou ~30s contínuos), nova linha com [MM:SS].
- NÃO identifique nem numere os falantes — só o tempo e a fala.
- [inaudível] só quando realmente incompreensível.
- NUNCA repita a mesma frase várias vezes seguidas.
- Devolva apenas a transcrição.
```

A instrução de não identificar falantes não é enfeite: uma versão anterior
pedia isso e o modelo "degenerava em loop" tentando adivinhar interlocutores
em áudios longos. Se o caso de uso não exige saber quem falou, não peça.

## Transcrição — com diarização (só se a pessoa aceitou o risco na entrevista)

Diarização aumenta a chance de degeneração em blocos longos, então combine
sempre com blocos menores (~8min em vez de 15min) e redes de segurança mais
rígidas:

```
Transcreva INTEGRALMENTE este áudio de uma reunião de trabalho
em português (ou no idioma predominante do áudio, ~8 minutos).
- Fiel e completa, sem resumir, sem omitir, sem comentar.
- A cada troca de quem fala, nova linha no formato [MM:SS] Falante N: texto.
- Numere os falantes na ordem em que aparecem (Falante 1, Falante 2, ...) —
  não tente adivinhar nomes reais a partir da voz.
- Se não tiver certeza de quem está falando, use o mesmo número da fala
  anterior em vez de inventar um novo falante.
- [inaudível] só quando realmente incompreensível.
- NUNCA repita a mesma frase ou a mesma marcação de falante várias vezes
  seguidas.
- Devolva apenas a transcrição.
```

Redes de segurança adicionais quando diarização está ligada:

- **Contagem de trocas de falante por minuto**: se um bloco tiver uma
  quantidade de trocas muito acima do normal para uma conversa (ex.: mais de
  ~20 trocas por minuto), é sinal de que o modelo está "piscando" entre
  falantes em vez de transcrever de verdade — rejeite o bloco e tente de
  novo, ou escale para um modelo mais capaz.
- Depois de combinar os blocos, deixe claro na interface que a numeração de
  falante (Falante 1, Falante 2...) **não é estável entre blocos** — o
  mesmo número pode não representar a mesma pessoa se o áudio for longo o
  bastante para gerar múltiplos blocos. Isso é uma limitação real da
  abordagem, não esconda isso da pessoa que vai revisar.
- Deixe a opção fácil de desligar (mesma variável de ambiente/config da
  entrevista) caso a taxa de rejeição de blocos fique alta em uso real —
  trate isso como algo a monitorar, não como decisão definitiva.

## Redes de segurança contra degeneração (aplicam-se com ou sem diarização)

Rejeite um bloco transcrito (e tente de novo ou escale de modelo) quando:

- Uma mesma linha se repete mais de ~12 vezes seguidas.
- Uma mesma frase com mais de ~40 caracteres se repete mais de ~10 vezes no
  total do bloco.
- O último timestamp do bloco está muito distante da duração real do bloco
  (sinal de cobertura ruim — o modelo parou de transcrever antes do fim).
- O motivo de parada da geração não é o esperado (ex.: cortou por limite de
  tokens em vez de terminar naturalmente).

Se um único bloco falhar depois de esgotar toda a escada de modelos, trate a
transcrição inteira como falha — uma transcrição parcial não é confiável o
suficiente para seguir para a extração.

## Extração (Claude) — resumo e itens sugeridos

Generalize o texto entre colchetes para o domínio real da instalação (não é
sempre uma reunião 1:1 gestor↔colaborador):

```
Você transforma a transcrição de [tipo de reunião/conversa] em dados
estruturados.

EXTRAIA:
1. "resumo": até 8 linhas — os principais assuntos, direto e sem
   julgamento, no mesmo idioma da transcrição.
2. "itens": cada combinado de ação citado na conversa. Para cada um:
   - "descricao": o que ficou combinado, objetivo e claro.
   - "responsavel_sugerido": o nome citado como responsável, ou null.

REGRAS: só o que está na transcrição — nada de inventar. Se não houver
combinado, "itens" é lista vazia.

FORMATO — APENAS este JSON, sem texto ao redor:
{"resumo": "...", "itens": [{"descricao": "...", "responsavel_sugerido": "..."}]}
```

### Parsing defensivo

- Remova cercas ```` ```json ```` antes de fazer o parse; se não tiver
  certeza de onde o JSON começa/termina, localize o primeiro `{` e o último
  `}` manualmente antes de tentar `JSON.parse`.
- Qualquer falha de parsing cai num objeto vazio
  (`{ resumo: '', itens: [] }`) em vez de derrubar o pipeline inteiro.
- Uma resposta cortada por limite de tokens é um erro explícito — peça
  reprocessamento em vez de aceitar um JSON truncado.

### Humano no loop, sempre

`responsavel_sugerido` é só um texto livre para *pré-selecionar* um
responsável real na interface (por aproximação de nome, por exemplo) — nunca
grave isso direto como `responsavel_id` no banco. Um item de ação só vira
registro definitivo depois que uma pessoa escolhe o responsável de verdade
(um ID de uma pessoa real do sistema) e confirma. Isso vale mesmo que a
sugestão da IA pareça óbvia — nome errado ou responsável errado custa mais
caro do que o segundo de atenção que a confirmação manual leva.

O campo de **observação** de cada item (nota livre, separada da descrição)
não faz parte do JSON de extração e não deve ser gerado pela IA — é
preenchido só pela pessoa na tela de revisão, se ela quiser registrar algum
contexto extra sobre o combinado.
