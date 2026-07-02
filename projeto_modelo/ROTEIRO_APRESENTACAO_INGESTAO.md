# Roteiro de Apresentação — Da Análise à Ingestão Perene
### Por que os 10 notebooks foram só metade do trabalho de engenharia

> **Tese da apresentação:** no Desafio 1 (`projetos_youtube_eng_dados`) nós
> agimos com a **pegada de cientista** — entender o negócio, explorar o dado,
> achar valor. Isso é legítimo e **faz parte** do trabalho do engenheiro. Mas o
> engenheiro de dados não para aí: ele precisa tornar aquilo **perene,
> confiável e automatizado**, aplicando as boas práticas que o curso inteiro
> (Encontros 1 a 6) apresentou. O Desafio 2 (`projeto_modelo`) é
> exatamente esse salto.

---

## Slide 1 — Capa
**Título:** Da Análise à Ingestão Perene
**Subtítulo:** Como transformar 10 notebooks de aula num pipeline que roda sozinho na VPS
- Disciplina: Linguagens de Programação para Engenharia de Dados — UNIFOR
- Desafio 2: evolução dos 10 pipelines do YouTube

**Fala:** "No último encontro a gente construiu 10 pipelines do YouTube e tirou análise de negócio de cada domínio. Hoje eu quero mostrar que aquilo foi só **metade** do trabalho — a metade do cientista. A outra metade, a do engenheiro, é o que vamos ver agora: transformar aquele código de aula num sistema que roda sozinho, todo dia, sem ninguém clicar."

---

## Slide 2 — Onde paramos: os 10 notebooks (Desafio 1)
- 10 domínios (finanças, esportes, geopolítica, saúde, e-commerce...), cada um com seu **Gold** específico.
- Pipeline **Bronze → Silver → Gold** executável, com fallback sintético e contrato Pandera.
- O aluno **colava os `VIDEO_IDS` na mão** e rodava o notebook uma vez para **explorar**.

**Fala:** "Repare na palavra-chave: *explorar*. A gente colava 2-3 IDs, rodava uma vez, olhava o resultado e interpretava o negócio. Isso é ótimo — e é insubstituível. Mas é uma postura de quem está **investigando o dado**, não de quem está **operando um sistema**."

---

## Slide 3 — A metáfora do curso: água × encanamento (Encontro 1)
> *"Se o dado é água, o engenheiro constrói o encanamento; o cientista analisa o que a água revela. Sem encanamento, não há análise."* — Encontro 1

| | Cientista de Dados | Engenheiro de Dados |
|---|---|---|
| Pergunta-chave | "O que este dado revela?" | "Como entrego este dado confiável, **na hora certa**?" |
| Nos 10 notebooks | o Gold por domínio | (ainda faltava) |

**Fala:** "Essa frase é do nosso primeiro encontro. Nos 10 notebooks a gente fez muito a parte de cima — *o que a água revela*. Mas a parte de baixo, o **encanamento que entrega a água confiável e na hora certa**, a gente só esboçou. O Desafio 2 é construir esse encanamento."

---

## Slide 4 — O que faltava: as perguntas do engenheiro
O notebook responde "o que o dado diz". Faltava responder:
1. **Quem cola os IDs todo dia?** (e se forem 30 canais?)
2. **Como eu sei o que já é novo** desde ontem, sem reprocessar tudo?
3. **Se eu rodar duas vezes, eu duplico os dados?**
4. **Quem garante que isso roda às 6h da manhã** sem eu abrir o VS Code?

**Fala:** "São quatro perguntas que o notebook simplesmente não responde — porque ele foi feito para rodar uma vez, com supervisão humana. Em produção, ninguém vai colar ID na mão nem ficar olhando célula rodar. Essas quatro perguntas viram os **quatro pilares** do nosso ingestor."

---

## Slide 5 — Citação: o que torna isto "engenharia" (Encontro 1)
> *"Um script roda uma vez; um pipeline roda muitas — de forma confiável."*

Os quatro critérios que o E1 já tinha cravado:
- **Idempotência** — rodar duas vezes dá o mesmo resultado, sem duplicar.
- **Tratamento de erro** — fonte malformada falha de forma clara, não corrompe em silêncio.
- **Rastreabilidade** — conto quantas linhas entraram, saíram e por que cada uma foi descartada.
- **Observabilidade** — o pipeline registra o que fez (logs) para eu confiar sem assistir.

**Fala:** "O bonito é que a régua já estava dada lá no Encontro 1. A gente não vai inventar nada novo — vamos **honrar** esses quatro critérios no código. Guardem essas quatro palavras: elas vão reaparecer em cada pilar."

---

## Slide 6 — O Desafio 2: ingestão perene
`projeto_modelo/` — a continuação natural dos 10 notebooks.

| | Desafio 1 (notebooks) | Desafio 2 (ingestor) |
|---|---|---|
| Entrada | cola `VIDEO_IDS` na mão | **descobre** sozinho (YouTube Data API) |
| Execução | roda 1 vez, supervisionado | roda **perene**, agendado |
| Estado | nenhum (começa do zero) | **SQLite**: watermark + idempotência |
| Postura | cientista explora | engenheiro **opera** |
| Entrega | notebook didático | **projeto-script** pronto p/ VPS |

**Fala:** "Mesma matéria-prima — transcrição do YouTube, mesmo Bronze/Silver/Gold, mesmo contrato Pandera. O que muda é tudo que está **em volta** do pipeline. É a diferença entre saber cozinhar um prato e abrir um restaurante que serve aquele prato todo dia."

---

## Slide 7 — Os 4 pilares (visão geral)
1. **Descoberta automática** — acabou colar ID; a API encontra os vídeos novos.
2. **Incrementalidade (watermark)** — só processa o que é novo desde a última vez.
3. **Idempotência** — rodar N vezes não duplica; legenda corrigida reprocessa.
4. **Agendamento por negócio** — cada canal no seu ritmo (guerra: 15min; review: 6h).

**Fala:** "Quatro pilares. Cada um responde uma das quatro perguntas do slide anterior, e cada um materializa um conceito que vimos em algum encontro do curso. Vamos um por um."

---

## Slide 8 — Pilar 1: Descoberta automática + estratégia de cota
- Sai de `VIDEO_IDS = [...]` para o canal inteiro, via **YouTube Data API v3**.
- **Decisão de engenharia (cota):** a API dá 10.000 unidades/dia.
  - ❌ `search.list` = **100 unidades** por chamada (caro!).
  - ✅ `channels.list` → playlist de uploads → `playlistItems.list` = **1 unidade**.
- Resultado: um canal custa **~2 unidades/ciclo** → dá para rodar dezenas de canais o dia todo dentro da cota grátis.

**Fala:** "Aqui já aparece a cabeça de engenheiro: não basta *funcionar*, tem que funcionar **dentro de uma restrição de custo**. A escolha de `playlistItems` em vez de `search` é a diferença entre rodar 100 canais ou estourar a cota antes do almoço."

---

## Slide 9 — Pilar 2: Incrementalidade (watermark) — eco do Encontro 4
```python
watermark = store.get_watermark(channel_id)   # maior publishedAt já processado
videos = disc.descobrir(channel_id, watermark, ...)  # só o que veio DEPOIS
```
> *"Em escala, o ganho não vem de processar mais rápido — vem de processar menos. Não processe o que você não precisa ler."* — Encontro 4

- Guarda o **maior `publishedAt`** por canal. No próximo ciclo, ignora tudo que é ≤ watermark.
- É o **particionamento e a leitura incremental** do E4, aplicados ao tempo.

**Fala:** "Lembram do Encontro 4? *Processe menos.* O watermark é isso: em vez de reprocessar o canal inteiro toda vez, eu guardo onde parei e só pego o que é novo. É barato, é incremental, e é o que permite rodar de 15 em 15 minutos sem peso."

---

## Slide 10 — Pilar 3: Idempotência + contrato — eco dos Encontros 1 e 5
```python
h = content_hash(texto_total)             # sha256 da transcrição
if store.ja_ingerido(video_id, h):        # já vi e não mudou?
    pulados += 1; continue                # IDEMPOTÊNCIA: não duplica
df = _bronze_silver(...)                   # senão: Bronze→Silver
SILVER_SCHEMA.validate(df)                 # contrato Pandera (mesmo do Desafio 1)
```
> *"Validar é metade do trabalho. A outra metade é decidir o que fazer com o que falhou."* — Encontro 5

- `content_hash` detecta **legenda corrigida** → reprocessa só esse vídeo.
- Falha de schema → vídeo vai para **FAILED** (quarentena do E5), não derruba o lote.

**Fala:** "Dois encontros num slide só. A idempotência é o critério do E1 — rodar duas vezes não duplica. E o tratamento do que falha é o E5 — eu não jogo fora em silêncio, eu marco como FAILED e registro o porquê. O `content_hash` é o detalhe esperto: se o canal corrige a legenda, o hash muda e só aquele vídeo reprocessa."

---

## Slide 11 — Pilar 4: Agendamento por negócio — eco do Encontro 6
```yaml
# config/canais.yaml — frequência é decisão de NEGÓCIO, não de código
- nome: "Geopolitica"; intervalo_min: 15    # guerra muda em minutos
- nome: "Financas";    intervalo_min: 60    # macro muda em horas
- nome: "Reviews";     intervalo_min: 360   # gadget não tem urgência → 6h
```
> *"Um agendador diz QUANDO. Um script roda uma vez; um pipeline roda sempre, sozinho e confiável."* — Encontro 6

- **APScheduler** agenda cada canal no seu intervalo, declarado em YAML (sem tocar no código).
- `--once` (1 ciclo), perene (agendado), `--status` (resumo do estado).

**Fala:** "O Encontro 6 separou *agendador* de *orquestrador*. Aqui estamos na camada do agendador: cada canal tem seu ritmo, e esse ritmo é uma **decisão de negócio** que mora num YAML — notícia de guerra eu quero de 15 em 15 minutos, review de celular pode esperar 6 horas. Mudar isso não exige mexer numa linha de Python."

---

## Slide 12 — A estrutura completa: um ciclo de ingestão
```
┌─ descobrir (incremental, via watermark)
│     └─ para cada vídeo NOVO e NÃO-ingerido:
│           captar transcrição ............ BRONZE
│           limpar + contrato Pandera ..... SILVER  → parquet particionado
│           atualizar estado SQLite ....... (idempotência)
├─ rodar analítico do domínio ............. GOLD    → densidade de termos
└─ avançar o watermark do canal
```
Saída — **lakehouse local** particionado:
```
datalake/
├── control/ingestion.db                  # estado (fonte da verdade)
├── silver/dominio=<x>/dt=<data>/*.parquet # Parquet particionado (E4)
└── gold/dominio=<x>/densidade_termos.parquet
```

**Fala:** "Essa é a fase completa. Note que o Bronze→Silver→Gold é o **mesmo** dos notebooks — a gente não jogou nada fora. O que entrou foi a casca: descoberta incremental antes, atualização de estado no meio, avanço do watermark no fim. E o Silver agora é **particionado** por domínio e data, exatamente o que o E4 ensinou."

---

## Slide 13 — O estado é a fonte da verdade (SQLite)
`ingestion_state` responde as 3 perguntas do engenheiro:

| Pergunta | Como o estado responde |
|---|---|
| O que eu já ingeri? | tabela `ingestion_state` (PK = `video_id`) |
| O que mudou desde a última vez? | `channel_watermark` (maior `publishedAt`) |
| Como evito duplicar/corromper? | UPSERT idempotente + `content_hash` |

- Status de cada vídeo: `DISCOVERED → INGESTED → FAILED/SKIPPED`.
- `--status` lê isso e dá **observabilidade** (critério do E1) sem abrir o banco.

**Fala:** "Sem estado, não há incrementalidade nem idempotência — você sempre começaria do zero. Esse SQLite é o coração do sistema: é ele que sabe o que já foi feito. É barato, é um arquivo só, zero setup, e responde as três perguntas que definem um engenheiro de dados."

---

## Slide 14 — Como tudo amarra no curso (E1 → E6)
| Encontro | Conceito | Onde aparece no ingestor |
|---|---|---|
| **E1** Fundamentos | idempotência, erro, rastreabilidade, observabilidade | os 4 critérios, em todo o `state.py` |
| **E2** pandas | DataFrame, limpeza | camada Silver (`_bronze_silver`) |
| **E3** SQL | agregação, JOIN, window | Gold em DuckDB |
| **E4** Escala | Parquet, particionamento, "processe menos" | watermark + `silver/dominio=/dt=` |
| **E5** Qualidade | Pandera, data contract, quarentena | `SILVER_SCHEMA` + status FAILED |
| **E6** Orquestração | agendar, repetir, reprocessar | APScheduler + `canais.yaml` |

**Fala:** "Esse slide é o resumo da tese. A ingestão perene não é um tópico novo — é o **curso inteiro convergindo num só artefato**. Cada encontro deixou uma peça, e o ingestor é onde todas elas se encaixam. Por isso ele é a continuação natural dos 10 notebooks."

---

## Slide 15 — Rodar perene de verdade: a VPS (systemd)
```bash
sudo cp youtube_ingestor/scripts/ingestor.service /etc/systemd/system/
sudo systemctl enable --now ingestor      # sobe no boot, reinicia se cair
journalctl -u ingestor -f                 # logs em tempo real (observabilidade)
```
- `Restart=always` → se o processo morre, o systemd ressuscita.
- Roda **sem o seu computador ligado** — é o "tirar o humano da cadeira" do E6.

**Fala:** "O último passo é o que torna a palavra *perene* verdadeira: sai da minha máquina e vai para um servidor que liga sozinho, reinicia sozinho e registra tudo. Aqui o pipeline finalmente roda **sem mim**. É a fronteira entre 'um script que eu rodo' e 'um sistema que existe'."

---

## Slide 16 — Demo
1. `python -m youtube_ingestor.scheduler --once` → 1 ciclo por canal (modo demo, sem API key).
2. `--status` → mostra `{'INGESTED': N}`.
3. Rodar `--once` de novo → o watermark avançou; mostra a incrementalidade em ação.
4. Abrir o Parquet Silver → ver as partições `dominio=/dt=` e a coluna do contrato.

**Fala (demo ao vivo):** "Modo demo roda sem chave de API e sem gastar cota — dados sintéticos deterministas. Vou rodar uma vez, mostrar o estado, rodar de novo e a gente vê o watermark fazendo seu trabalho."

---

## Slide 17 — Fechamento
- Os 10 notebooks foram a **pegada cientista**: entender o negócio. Necessária e insubstituível.
- A ingestão perene é a **pegada engenheiro**: tornar aquilo confiável, incremental e automático.
- As duas **não competem** — o engenheiro entende o negócio *e* constrói o encanamento.

**Frase de efeito:** "O notebook responde 'o que o dado diz'. O ingestor responde 'como esse dado chega, confiável, todo dia, sem mim'. Engenharia de dados é dominar as duas perguntas."
</content>
