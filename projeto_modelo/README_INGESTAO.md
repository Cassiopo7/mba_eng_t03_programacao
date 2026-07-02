# projeto_modelo — Ingestão Perene de Dados do YouTube

**Desafio 2** da disciplina. Evolução do desafio dos 10 notebooks: sai da coleta manual (colar ID) para uma
**ingestao automatizada, incremental e governada**, pronta para rodar perene
numa VPS. Postura de **engenheiro de dados**, nao de cientista.

## Conteudo

| Item                                  | O que e                                                                                                   |
| ------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `00_ingestao_perene_DIDATICO.ipynb` | Notebook que ENSINA os 4 pilares (descoberta, watermark, idempotencia, agendamento) com codigo executavel |
| `youtube_ingestor/`                 | Projeto-script de PRODUCAO, testado, pronto para a VPS                                                    |

## Os 4 pilares

1. **Descoberta automatica** (YouTube Data API v3) — sem colar ID. Com estrategia
   de cota: `playlistItems.list` (1 unidade) em vez de `search.list` (100).
2. **Incrementalidade (watermark)** — guarda o maior `publishedAt` por canal;
   so processa o que e novo.
3. **Idempotencia** — `ingestion_state` (SQLite) com `video_id` + `content_hash`;
   rodar N vezes nao duplica, e legenda corrigida dispara reprocessamento.
4. **Agendamento por negocio** — APScheduler; cada canal no seu intervalo
   (guerra: 15min, financas: 60min, review: 6h), definido em `canais.yaml`.

## Estrutura do projeto-script

```
youtube_ingestor/
├── config/canais.yaml      # canais + intervalo por negocio (declarativo)
├── ingestor/
│   ├── discovery.py        # Data API v3 + estrategia de cota (+ modo demo)
│   ├── state.py            # SQLite: watermark + idempotencia (fonte da verdade)
│   ├── transcript.py       # captacao da legenda (fallback sintetico)
│   └── pipeline.py         # Bronze -> Silver(Pandera) -> Gold(DuckDB)
├── scripts/
│   ├── resolver_handle.py  # @handle -> channel_id (UC...) p/ colar no canais.yaml
│   └── ingestor.service    # unit do systemd p/ rodar perene na VPS
├── scheduler.py            # entrypoint APScheduler (--once / --status / perene)
├── requirements.txt
└── .env.example
```

Descobrir o `channel_id` a partir do `@handle` (gasta 1 unidade de cota):

```bash
export YOUTUBE_API_KEY=...        # veja .env.example
python -m youtube_ingestor.scripts.resolver_handle @nomedocanal
```

## Como rodar

```bash
pip install -r youtube_ingestor/requirements.txt

python -m youtube_ingestor.scheduler --once     # 1 ciclo por canal e sai
python -m youtube_ingestor.scheduler            # perene (agenda por canal)
python -m youtube_ingestor.scheduler --status   # resumo do estado acumulado
```

`modo_demo: true` (padrao em `canais.yaml`) roda **sem API key e sem gastar
cota**, com dados sinteticos deterministicos — a aula inteira funciona offline.
Para producao: `modo_demo: false`, preencha `channel_id` reais e
`YOUTUBE_API_KEY` no `.env` (copie de `.env.example`).

## Saida (lakehouse local)

```
datalake/
├── control/ingestion.db                 # estado SQLite (watermark + idempotencia)
├── silver/dominio=<x>/dt=<data>/*.parquet
└── gold/dominio=<x>/densidade_termos.parquet
```

**O que evoluiu em relacao aos notebooks (Desafio 1):**

- **Particionamento.** Os notebooks gravavam arquivo unico
  (`silver/transcricoes.parquet`). Aqui o Silver e particionado por
  `dominio=` e `dt=` (data) — porque agora a ingestao e continua e roda para
  varios canais/dominios, e particionar e o que torna a leitura incremental
  barata. O contrato Silver (Pandera, incluindo `n_palavras`) e o MESMO dos
  notebooks.
- **Gold generico.** Cada notebook tinha um Gold proprio (10 tecnicas: Polars
  Lazy, `ROW_NUMBER`/`QUALIFY`, anti-join etc.). Para a ingestao perene
  usamos um Gold unico e simples — densidade de termos do `vocabulario` do
  canal. A ideia e cada turma especializar `_gold()` em `pipeline.py` com a
  tecnica do seu notebook de referencia.

## Rodar como servico na VPS (systemd)

Ja existe um unit pronto em `youtube_ingestor/scripts/ingestor.service`. Ajuste
`WorkingDirectory`/`ExecStart`/`User` para onde voce clonou o projeto e instale:

```bash
sudo cp youtube_ingestor/scripts/ingestor.service /etc/systemd/system/ingestor.service
sudo systemctl daemon-reload
sudo systemctl enable --now ingestor
journalctl -u ingestor -f
```
