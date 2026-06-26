# Masterclass Pratica: 10 Pipelines Reais com YouTube API

**Disciplina:** Linguagens de Programacao para Engenharia de Dados
**Instituicao:** UNIFOR · **Professor:** Cassio Pinheiro
**Formato:** atividade pratica em VS Code (extensao Jupyter) com Python (~3h)

## O que mudou em relacao ao material original

O material original entregava celulas apenas com comentarios `# TODO`, o que trava
o aluno na primeira execucao e depende de IDs de video com legenda disponivel.
Esta versao reescreve os 10 projetos com:

- **Pipeline Bronze -> Silver -> Gold executavel de ponta a ponta** (nada de `# TODO`
  bloqueante; os antigos TODOs viraram "Desafios de extensao" opcionais ao final).
- **Fallback sintetico determinista** (`seed=42`): se nao houver IDs validos ou
  legendas, o pipeline gera dados e roda igual em qualquer maquina. A aula nunca para.
- **API correta da `youtube-transcript-api` v1.x** (`YouTubeTranscriptApi().fetch()`),
  e nao o metodo estatico antigo `.get_transcript()`.
- **Contrato de dados real com Pandera** (`pandera.pandas`, API atual) + area de
  **quarentena** para descartes.
- **Camada Gold especifica por dominio**: DuckDB (CROSS JOIN, LIKE composto,
  CASE WHEN, JOIN com dimensao, anti-join de lacunas, window function `ROW_NUMBER`
  + `QUALIFY`) e Polars Lazy (particionamento, filtro de janela de retencao).

## Os 10 notebooks

| # | Arquivo | Dominio | Tecnica Gold em destaque |
|---|---|---|---|
| 1 | `01_financas_e_mercado.ipynb` | Financas | Densidade de termos por minuto (CROSS JOIN) |
| 2 | `02_esportes_e_futebol.ipynb` | Futebol | Particionamento por clima (Polars Lazy) |
| 3 | `03_geopolitica_e_noticias.ipynb` | Geopolitica | Filtro composto por pais (LIKE/JOIN) |
| 4 | `04_tecnologia_e_ia.ipynb` | Tech/IA | Classificacao por camada (CASE WHEN) |
| 5 | `05_marketing_digital_e_concorrencia.ipynb` | Marketing | Janela de retencao <=60s (Polars Lazy) |
| 6 | `06_saude_publica_e_bem_estar.ipynb` | Saude | JOIN com dimensao de substancias |
| 7 | `07_entretenimento_e_pop_culture.ipynb` | Entretenimento | Balanco de sentimento (SUM condicional) |
| 8 | `08_e_commerce_e_reviews.ipynb` | Reviews | Ranking de defeitos (`ROW_NUMBER`/`QUALIFY`) |
| 9 | `09_educacao_e_edtech.ipynb` | Educacao | Lacunas da ementa (anti-join) |
| 10 | `10_seguranca_publica_e_crimes.ipynb` | Seguranca | Agregacao de ocorrencias por crime |

## Pre-requisitos (uma unica vez)

1. Instale o **VS Code** com as extensoes **Python** e **Jupyter**.
2. Na pasta do projeto, crie um ambiente virtual e instale as dependencias:
   ```bash
   python -m venv .venv
   source .venv/bin/activate        # Windows: .venv\Scripts\activate
   pip install -r requirements.txt
   ```

## Como usar em sala

1. Cada grupo escolhe **um** notebook (um dominio) e o abre no VS Code.
2. Seleciona o **kernel** do `.venv` (canto superior direito do notebook).
3. **Opcional mas recomendado:** busca 2-3 videos reais do tema no YouTube, copia
   os IDs (trecho apos `v=` na URL) e cola na lista `VIDEO_IDS` da camada Bronze.
   Sem isso, o notebook roda com dados sinteticos do mesmo jeito.
4. Executa Bronze -> Silver -> Gold em sequencia e interpreta o resultado de negocio.
5. Quem terminar antes ataca os **Desafios de extensao** ao final do notebook.

## Dependencias

`pandas`, `polars`, `duckdb`, `pandera`, `youtube-transcript-api`, `pyarrow`
— declaradas em `requirements.txt` (instaladas uma vez no `.venv`, nao mais celula a celula).

## Estrutura de saida (data lakehouse local)

```
./datalake/
  bronze/    transcricoes brutas (.parquet)
  silver/    transcricoes validadas pelo contrato
  silver/_quarentena/  descartes isolados
  gold/      tabela analitica final do projeto
```
