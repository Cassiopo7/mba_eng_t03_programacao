# Contrato de Trabalho Final
## Linguagens de Programação para Engenharia de Dados — UNIFOR

**Professor:** Cassio Pinheiro · [LinkedIn](https://www.linkedin.com/in/cassio-pinheiro-9baa7939/)  
**Curso:** Pós-Graduação Lato Sensu — Especialização em Engenharia de Dados  
**Modalidade:** trabalho em grupo · entrega em produção na VPS · observação de 7 dias

---

## 1. O que é este trabalho

Este é o **artefato final da disciplina**: cada grupo transforma uma das temáticas
de `projetos_youtube_eng_dados/` em um **sistema de dados completo** — da captura
automatizada até a entrega de **insights para um público de negócio**.

Não basta ter um notebook que roda uma vez. O trabalho exige **postura de engenheiro
de dados**: pipeline perene, confiável, observável e publicado num ambiente real.

> **Fio condutor do curso:** nos Encontros 1 a 6 vocês construíram o vocabulário
> (ETL, pandas, SQL, escala, qualidade, orquestração). No Desafio 1 (`projetos_youtube_eng_dados/`)
> exploraram o dado como cientistas. No Desafio 2 (`projeto_modelo/`) viram como
> operar um ingestor na VPS. **Agora é a entrega integrada de tudo isso.**

---

## 2. Escolha da temática

Cada grupo escolhe **uma** das dez temáticas abaixo. Duas equipes **não podem**
usar o mesmo domínio — a escolha é por ordem de confirmação com o professor.

| # | Domínio | Notebook de referência | Pergunta de negócio (exemplo) |
|---|---|---|---|
| 1 | Finanças & Mercado | `01_financas_e_mercado.ipynb` | Com que frequência termos macro (juros, inflação, recessão) aparecem nos podcasts financeiros? |
| 2 | Esportes & Futebol | `02_esportes_e_futebol.ipynb` | Como o clima e o contexto da partida aparecem na narrativa esportiva? |
| 3 | Geopolítica & Notícias | `03_geopolitica_e_noticias.ipynb` | Quais países e temas dominam a cobertura em canais de notícia? |
| 4 | Tecnologia & IA | `04_tecnologia_e_ia.ipynb` | Como os criadores classificam ferramentas e camadas da stack de IA? |
| 5 | Marketing Digital | `05_marketing_digital_e_concorrencia.ipynb` | O que os canais de marketing enfatizam nos primeiros 60 segundos? |
| 6 | Saúde Pública | `06_saude_publica_e_bem_estar.ipynb` | Quais substâncias e hábitos são mais mencionados em conteúdo de saúde? |
| 7 | Entretenimento | `07_entretenimento_e_pop_culture.ipynb` | Qual o balanço de sentimento em torno de obras e celebridades? |
| 8 | E-commerce & Reviews | `08_e_commerce_e_reviews.ipynb` | Quais defeitos e elogios mais se repetem em reviews de produtos? |
| 9 | Educação & EdTech | `09_educacao_e_edtech.ipynb` | Quais tópicos da ementa ficam de fora do que os canais ensinam? |
| 10 | Segurança Pública | `10_seguranca_publica_e_crimes.ipynb` | Como diferentes tipos de ocorrência aparecem na cobertura? |

O notebook escolhido é **ponto de partida**, não entrega final. Vocês devem evoluir
a lógica de Gold, automatizar a ingestão e expor o resultado em produção.

---

## 3. Carta do projeto (obrigatória)

Antes de escrever código em produção, cada grupo entrega um arquivo
`docs/CARTA_DO_PROJETO.md` com os campos abaixo **preenchidos e assinados por todos
os integrantes** (nome + e-mail).

### 3.1 Identidade

| Campo | O que preencher |
|---|---|
| **Nome do projeto** | Título comercial — como vocês apresentariam para um gestor |
| **Tema / domínio** | Um dos dez domínios da tabela acima |
| **Equipe** | Nomes, papéis (eng. dados, qualidade, deploy, produto/negócio) |
| **Data de início** | Quando o grupo formalizou o projeto |

### 3.2 Problema e propósito

| Campo | O que preencher |
|---|---|
| **Problema** | Qual dor concreta existe hoje? Quem sofre com ela? Por que planilha/manual não resolve? |
| **Propósito** | O que o sistema passa a fazer automaticamente que antes não existia? |
| **Público-alvo** | Quem consome o insight (analista, gestor, jornalista, investidor…)? |
| **Hipótese de valor** | Uma frase testável: *"Se monitorarmos X, conseguiremos decidir Y em Z tempo."* |

### 3.3 Escopo técnico

| Campo | O que preencher |
|---|---|
| **Fontes de dados** | Canais YouTube (handles/IDs), vocabulário de termos, idiomas de legenda |
| **Frequência de ingestão** | Intervalo por canal e justificativa de negócio (minutos/horas) |
| **Métrica principal (KPI)** | O número ou ranking que o dashboard deve destacar |
| **Perguntas analíticas** | Pelo menos 3 perguntas que o Gold responde |
| **Fora de escopo** | O que vocês **não** vão fazer (evita creep) |

### 3.4 Critérios de sucesso

| Campo | O que preencher |
|---|---|
| **Definição de pronto** | Checklist objetivo: pipeline roda sozinho, dashboard publicado, 7 dias sem intervenção manual |
| **Riscos** | API sem cota, canal sem legenda, VPS indisponível — e o plano B de cada um |

---

## 4. Arquitetura obrigatória

O sistema deve seguir o padrão **lakehouse em camadas** ensinado na disciplina,
evoluindo o `projeto_modelo/`:

```
<seu_projeto>/
├── docs/
│   ├── CARTA_DO_PROJETO.md
│   ├── ARQUITETURA.md
│   ├── RUNBOOK.md              # como operar, reiniciar, ler logs
│   └── RELATORIO_7_DIAS.md     # artefato pós-observação (seção 7)
├── config/
│   └── canais.yaml             # canais, domínio, intervalo, vocabulário
├── ingestor/                   # pode partir de projeto_modelo/youtube_ingestor/
│   ├── discovery.py
│   ├── state.py
│   ├── transcript.py
│   └── pipeline.py             # Bronze → Silver → Gold (customizado)
├── app/
│   └── streamlit_app.py        # dashboard de insights
├── datalake/                   # gerado em runtime (não versionar dados brutos)
│   ├── control/ingestion.db
│   ├── bronze/
│   ├── silver/
│   │   └── _quarentena/
│   └── gold/
├── deploy/
│   ├── ingestor.service        # systemd
│   └── streamlit.service       # systemd (ou equivalente documentado)
├── requirements.txt
├── .env.example                # sem segredos reais
└── README.md                   # como clonar, configurar e rodar
```

### 4.1 Camadas — requisitos mínimos

| Camada | Obrigatório | Referência no curso |
|---|---|---|
| **Bronze** | Captura bruta de transcrições (API ou modo demo documentado) | Desafio 1 + `projeto_modelo` |
| **Silver** | Contrato Pandera, quarentena de descartes, particionamento `dominio=` / `dt=` | Encontro 5 |
| **Gold** | Transformação analítica **específica do domínio** (técnica do notebook de referência) | Desafio 1 |
| **Controle** | SQLite com watermark + idempotência (`video_id`, `content_hash`) | `projeto_modelo` |
| **Agendamento** | Ingestão perene (APScheduler ou orquestrador documentado) | Encontro 6 |
| **Apresentação** | Dashboard Streamlit lendo Gold (Parquet/DuckDB) | Livre (sugerido) |

### 4.2 Princípios de engenharia (não negociáveis)

Estes quatro critérios foram cravados no Encontro 1 e **serão avaliados**:

1. **Idempotência** — rodar o pipeline N vezes não duplica dado.
2. **Tratamento de erro** — falha de fonte vai para log/quarentena, não corrompe em silêncio.
3. **Rastreabilidade** — é possível saber quantas linhas entraram, validaram e foram descartadas.
4. **Observabilidade** — logs permitem entender o que o sistema fez sem abrir o VS Code.

---

## 5. Dashboard de insights (Streamlit)

O dashboard é a **porta de saída do produto de dados** para o público definido na Carta.

### Requisitos mínimos

- **Página inicial** com nome do projeto, problema, propósito e última atualização dos dados.
- **Pelo menos 3 visualizações** ligadas às perguntas analíticas da Carta (gráficos, tabelas ou rankings).
- **KPI em destaque** — o indicador principal que justifica o projeto.
- **Metadados de confiabilidade** — data da última ingestão, quantidade de vídeos no período, aviso se estiver em modo demo.
- **URL pública na VPS** — acessível durante os 7 dias de observação e na apresentação final.

### Sugestões (não obrigatórias, mas pontuam)

- Filtro por período ou canal.
- Download de amostra do Gold (CSV/Parquet).
- Seção "Como interpretar" — 2–3 frases em linguagem de negócio.

---

## 6. Período de observação — 7 dias

Após o deploy na VPS, inicia-se um **período contínuo de 7 dias corridos** em que o
sistema deve rodar **sem intervenção manual na ingestão**.

### 6.1 O que conta como "observação"

- O ingestor dispara nos intervalos configurados em `canais.yaml`.
- Novos vídeos (quando existirem) entram no lakehouse.
- O dashboard reflete dados atualizados (atualização automática ou documentada).
- Falhas são registradas em log — não precisam ser zero, mas precisam ser **explicadas**.

### 6.2 O que o grupo deve fazer durante os 7 dias

| Dia | Ação mínima |
|---|---|
| D0 | Deploy concluído; registrar horário de início em `RELATORIO_7_DIAS.md` |
| D1–D6 | Checagem diária (5–10 min): serviço ativo, última execução, dashboard no ar |
| D7 | Coleta de métricas finais; redação do relatório |

### 6.3 Métricas a registrar

Preencher no `docs/RELATORIO_7_DIAS.md`:

- Total de ciclos de ingestão executados.
- Vídeos descobertos / processados / em quarentena.
- Erros por tipo (sem legenda, API, validação Pandera…).
- Evolução do KPI principal ao longo da semana.
- Incidentes e como foram resolvidos.
- Capturas de tela do dashboard (início, meio e fim do período).

---

## 7. Artefato final — o agrupado

A entrega não é um arquivo solto: é um **pacote completo** que prova que o pipeline
funcionou em produção. Tudo abaixo compõe o artefato final.

### 7.1 Repositório Git

- Repositório público ou privado (link entregue ao professor).
- Histórico de commits coerente (não um único commit no último dia).
- `.gitignore` correto: sem `datalake/`, `.env`, `.venv/`.

### 7.2 Documentação

| Documento | Conteúdo |
|---|---|
| `README.md` | Visão geral, setup, comandos, URL do dashboard |
| `docs/CARTA_DO_PROJETO.md` | Seção 3 completa |
| `docs/ARQUITETURA.md` | Diagrama do fluxo + decisões técnicas |
| `docs/RUNBOOK.md` | Start/stop, logs, troubleshooting |
| `docs/RELATORIO_7_DIAS.md` | Métricas e narrativa do período de observação |

### 7.3 Código em produção

- Ingestor e Streamlit rodando na VPS no momento da avaliação.
- `config/canais.yaml` com canais reais do domínio escolhido.
- Gold customizado (não apenas o template genérico de densidade de termos).

### 7.4 Apresentação final (15 min por grupo)

1. **Problema e propósito** (2 min) — Carta do Projeto em linguagem de negócio.
2. **Demo ao vivo** (5 min) — dashboard na VPS + um trecho do pipeline.
3. **Engenharia** (5 min) — camadas, qualidade, idempotência, o que aprendeu.
4. **7 dias em produção** (3 min) — números do relatório, surpresas, próximos passos.

---

## 8. Cronograma sugerido

| Fase | Prazo sugerido | Entrega |
|---|---|---|
| **F0 — Formalização** | até 3 dias após o Encontro 6 | Domínio reservado + `CARTA_DO_PROJETO.md` |
| **F1 — Pipeline local** | +1 semana | Bronze → Silver → Gold rodando localmente |
| **F2 — Ingestão perene** | +1 semana | `projeto_modelo` adaptado + agendamento |
| **F3 — Dashboard** | +3 dias | Streamlit local consumindo Gold |
| **F4 — Deploy VPS** | +3 dias | Serviços no ar; início dos 7 dias |
| **F5 — Observação** | 7 dias corridos | Monitoramento + `RELATORIO_7_DIAS.md` |
| **F6 — Entrega final** | após D7 | Repositório + apresentação |

*Datas exatas serão confirmadas pelo professor no fechamento da disciplina.*

---

## 9. Critérios de avaliação

| Critério | Peso | O que o professor verifica |
|---|---:|---|
| **Carta do projeto** | 10% | Problema claro, propósito, KPI e escopo bem definidos |
| **Pipeline Bronze → Gold** | 25% | Camadas corretas, Gold do domínio, particionamento |
| **Qualidade e confiabilidade** | 20% | Pandera, quarentena, idempotência, logs |
| **Produção na VPS** | 20% | Deploy estável, serviços systemd, `.env` fora do Git |
| **Dashboard / insights** | 15% | Streamlit útil, KPI, visualizações ligadas ao negócio |
| **Relatório dos 7 dias** | 10% | Métricas, incidentes, evidências, honestidade técnica |

### Descontos automáticos

- Dashboard fora do ar na apresentação: **−10%**.
- Repositório sem `RUNBOOK` ou `CARTA`: **−5%** cada.
- Segredo vazado no Git: **nota zero** na parte de deploy (repositório refeito).

### Bônus (até +5% no critério de pipeline)

- Gold com técnica avançada do notebook de referência (window function, anti-join, Polars lazy…).
- Alerta simples (e-mail/log) quando ingestão falha 3 vezes seguidas.
- Testes automatizados do contrato Silver.

---

## 10. Checklist de entrega

Marque antes de enviar o link final ao professor:

```
[ ] Domínio confirmado e único na turma
[ ] docs/CARTA_DO_PROJETO.md assinada por todos
[ ] Pipeline Bronze → Silver → Gold executável
[ ] Contrato Pandera + pasta _quarentena funcionando
[ ] Watermark e idempotência no SQLite
[ ] Agendamento perene configurado
[ ] Streamlit publicado na VPS com URL estável
[ ] 7 dias de observação concluídos
[ ] docs/RELATORIO_7_DIAS.md com métricas e prints
[ ] README + RUNBOOK + ARQUITETURA completos
[ ] .env fora do Git; .env.example atualizado
[ ] Apresentação de 15 min preparada
```

---

## 11. Referências do material da disciplina

| Material | Uso no trabalho final |
|---|---|
| `encontros/` (E1–E6) | Princípios de pipeline, SQL, escala, qualidade, orquestração |
| `projetos_youtube_eng_dados/` | Domínio, vocabulário, técnica Gold de referência |
| `projeto_modelo/` | Base do ingestor perene, `canais.yaml`, systemd |

---

*Contrato de Trabalho Final — UNIFOR Pós-Graduação em Engenharia de Dados · Prof. Cassio Pinheiro.*
