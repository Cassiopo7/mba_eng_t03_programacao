# Pacote Didático da Disciplina
## Linguagens de Programação para Engenharia de Dados — UNIFOR

**Professor:** Cassio Pinheiro · [LinkedIn](https://www.linkedin.com/in/cassio-pinheiro-9baa7939/)
**Curso:** Pós-Graduação Lato Sensu — Especialização em Engenharia de Dados
**Carga horária:** 24h · **6 encontros** · 19:00 às 22:30
**Datas:** 25/06, 26/06, 27/06, 09/07, 10/07 e 11/07/2026

---

## O que é este pacote

Material completo e autocontido dos **6 encontros** da disciplina. Cada encontro
entrega três artefatos **cronologicamente alinhados** entre si — a seção *N* do
documento corresponde à seção *N* do notebook e ao bloco de slides *N*:

- **`*_material.md`** — documento teórico enriquecido, com blocos "Pontos de Atenção" e "Conexão com a Prática";
- **`*_pratica.ipynb`** — notebook executável de ponta a ponta (validado, 0 erros), com saídas já embutidas;
- **`*_slides.pptx`** — apresentação com o tema institucional UNIFOR (azul/branco).

A disciplina é uma das **primeiras do curso** e parte de uma turma **heterogênea**
(nem todos vêm de TI). A progressão sai do conceitual/nivelador (E1) e avança,
encontro a encontro, até um pipeline orquestrado em produção (E6), sempre
ancorada nas **principais ferramentas e tendências do mercado** de Engenharia de
Dados (2025–2026).

---

## Estrutura do repositório

```
.
├── README.md
├── CONTRATO_TRABALHO_FINAL.md   # artefato final da disciplina
├── slidekit.py                  # utilitário de slides (tema UNIFOR)
├── encontros/                   # 6 encontros (material + prática + slides)
├── projetos_youtube_eng_dados/  # Desafio 1 — 10 pipelines Bronze→Gold
└── projeto_modelo/              # Desafio 2 — ingestão perene (base p/ VPS)
```

---

## Desafios práticos e trabalho final

| Etapa | Pasta | O que é |
|---|---|---|
| **Desafio 1** | `projetos_youtube_eng_dados/` | Masterclass com 10 domínios; cada notebook extrai transcrições do YouTube e gera Gold analítico (exploração, postura de cientista). |
| **Desafio 2** | `projeto_modelo/` | Evolução para ingestão perene: descoberta automática, watermark, idempotência e agendamento — pronto para rodar na VPS. |
| **Trabalho final** | `CONTRATO_TRABALHO_FINAL.md` | Cada grupo escolhe um domínio, monta pipeline completo (captura → Gold → dashboard Streamlit), publica na VPS e observa 7 dias em produção. |

**Progressão pedagógica:** Encontros 1–6 (fundamentos) → Desafio 1 (explorar o dado) → Desafio 2 (operar o pipeline) → Trabalho final (entrega integrada em produção).

---

## Mapa da disciplina

| # | Data | Encontro | Pasta | Ferramentas / tendências |
|---|---|---|---|---|
| 1 | 25/06 | **Fundamentos** — o que é Engenharia de Dados e por que programar | `encontros/Encontro 1 - Fundamentos/` | Python puro, ETL, DuckDB (opcional) |
| 2 | 26/06 | **pandas** — a tabela como objeto de programação | `encontros/Encontro 2 - pandas/` | pandas, Apache Arrow, Polars |
| 3 | 27/06 | **SQL** para Engenharia de Dados | `encontros/Encontro 3 - SQL/` | DuckDB, modelo estrela, dbt |
| 4 | 09/07 | **Processamento em escala** | `encontros/Encontro 4 - Processamento em Escala/` | Parquet/Arrow, chunking, Polars lazy, PySpark, Lakehouse |
| 5 | 10/07 | **Qualidade e confiabilidade de dados** | `encontros/Encontro 5 - Qualidade de Dados/` | Pandera, Great Expectations, data contracts |
| 6 | 11/07 | **Orquestração e o pipeline em produção** | `encontros/Encontro 6 - Orquestracao/` | DAGs, Airflow, Dagster, Prefect, CI/CD |

Cada pasta em `encontros/` contém os três artefatos do encontro (`*_material.md`,
`*_pratica.ipynb`, `*_slides.pptx`). Veja também `CONTRATO_TRABALHO_FINAL.md`
para a entrega final da disciplina.

> **Fio condutor único:** um `vendas.csv` sintético *deliberadamente sujo* (datas
> em formatos diferentes, valores ausentes/negativos, duplicata, espaços) que
> evolui ao longo do curso — limpo com Python puro (E1), refeito em pandas e
> persistido em Parquet (E2), consultado em SQL sobre um modelo estrela (E3),
> escalado para centenas de milhares de linhas (E4), validado e colocado em
> quarentena (E5) e, por fim, orquestrado como um DAG completo em produção (E6).

---

## Estrutura por encontro

**Encontro 1 — Fundamentos.** O que é Engenharia de Dados (× Ciência de Dados); o
dado como produto (bruto → confiável); por que programar; blocos do Python;
estruturas nativas (lista/dicionário/tupla/conjunto); primeiro mini-pipeline ETL
com rastreabilidade.

**Encontro 2 — pandas.** Por que pandas (vetorização × laços); DataFrame/Series e
leitura CSV/JSON/Parquet; seleção e filtragem (`loc`/`iloc`); limpeza; agregação
e `merge`; o pipeline do E1 refeito em pandas, salvando Parquet. *Tendência:*
backend Arrow e Polars.

**Encontro 3 — SQL.** Pensar em conjuntos; `SELECT`/agregação/`JOIN`; window
functions e CTEs; SQL embarcado com DuckDB sobre DataFrames e Parquet.
*Tendência:* DuckDB e dbt.

**Encontro 4 — Escala.** O problema da escala; formatos colunares (Parquet/Arrow);
particionamento e chunking; lazy evaluation (Polars); distribuído (PySpark);
qual ferramenta usar. *Tendência:* small-data engines e Lakehouse (Delta/Iceberg).

**Encontro 5 — Qualidade.** Dimensões de qualidade; validação programática com
Pandera; Great Expectations; data contracts; quarentena de dados ruins;
observabilidade e lineage. *Tendência:* shift-left data quality e contracts.

**Encontro 6 — Orquestração.** De script a DAG; dependências, agendamento,
retries; Airflow; orquestração asset-based (Dagster/Prefect); produção e CI/CD;
fechamento do curso (o pipeline E1→E6 como sistema). *Tendência:* orquestração
declarativa e "orchestration as code".

---

## Dependências dos notebooks

Todos os notebooks foram **validados executando de ponta a ponta (0 erros)** e são
**determinísticos**. O núcleo de cada um roda com bibliotecas leves e instaláveis;
frameworks pesados aparecem em células **opcionais** (try/except com fallback que
exibe o código), de modo que o notebook nunca quebra por falta deles.

```bash
# Núcleo (cobre o que de fato executa em todos os encontros)
pip install pandas pyarrow polars duckdb pandera numpy

# Opcionais (apenas ilustrativos no material — NÃO exigidos para rodar):
#   pyspark            (E4)
#   great_expectations (E5)
#   apache-airflow / prefect / dagster (E6)
```

| Encontro | Núcleo executável | Opcional (ilustrativo) |
|---|---|---|
| 1 | biblioteca padrão | duckdb |
| 2 | pandas, pyarrow | polars |
| 3 | duckdb, pandas | — |
| 4 | pandas, numpy, pyarrow, polars | pyspark |
| 5 | pandas, pandera | great_expectations |
| 6 | biblioteca padrão, pandas | airflow, prefect, dagster |

---

## Ordem sugerida de uso em sala (3h30 por encontro)

1. **Abertura (15 min)** — slides de capa, agenda e contrato pedagógico.
2. **Conceitos (60 min)** — slides de conteúdo + documento `.md` (aula expositiva).
3. **Mão na massa — parte 1 (45 min)** — primeiras seções do notebook.
4. **Intervalo.**
5. **Mão na massa — parte 2 (60 min)** — seções finais + pipeline do encontro.
6. **Fechamento (20 min)** — slide de síntese + exercícios; tarefa para o próximo encontro.

---

## Bibliografia (comum à disciplina)

- REIS, Joe; HOUSLEY, Matt. *Fundamentos de Engenharia de Dados*. O'Reilly / Alta Books, 2023.
- MCKINNEY, Wes. *Python para Análise de Dados*. 3ª ed. O'Reilly, 2022.
- KLEPPMANN, Martin. *Designing Data-Intensive Applications*. O'Reilly, 2017.

---

*Material de apoio — UNIFOR Pós-Graduação · Prof. Cassio Pinheiro.
Repositório: encontros, desafios práticos (`projetos_youtube_eng_dados/`, `projeto_modelo/`)
e contrato do trabalho final (`CONTRATO_TRABALHO_FINAL.md`).*
