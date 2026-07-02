# Linguagens de Programação para Engenharia de Dados
## Encontro 6 — Orquestração e o pipeline em produção

**Instituição:** Universidade de Fortaleza (UNIFOR)
**Curso:** Pós-Graduação Lato Sensu — Especialização em Engenharia de Dados
**Professor:** Cassio Pinheiro · [LinkedIn](https://www.linkedin.com/in/cassio-pinheiro-9baa7939/)
**Data:** 11/07/2026 · 19:00 às 22:30
**Carga horária da disciplina:** 24h

---

## Como ler este documento

Este é o material de fundamentação do **sexto e último encontro** da disciplina. Ele acompanha:

- o **notebook** `e6_pratica.ipynb` (onde rodamos código — cada seção do notebook é referenciada aqui com a marca 🔗);
- os **slides** `e6_slides.pptx` (para a aula expositiva).

Chegamos ao fim. Ao longo de cinco encontros construímos o vocabulário e o primeiro pipeline (E1), domamos o pandas (E2), exercitamos transformação e SQL (E3), enfrentamos a escala (E4) e blindamos o dado com qualidade, contratos e quarentena (E5). Em todos eles havia uma peça implícita: **alguém apertava "rodar"**. Hoje removemos essa pessoa do circuito. Vamos transformar o conjunto de scripts do curso em **um pipeline que roda sozinho, na hora certa, em ordem, com tentativa de recuperação automática e com voz própria quando algo falha**. Esse é o trabalho de **orquestração** — e é o que separa um experimento de notebook de um sistema de dados em produção.

> **Combinado pedagógico:** orquestração não é uma ferramenta nova para decorar; é a formalização de princípios que plantamos desde o Encontro 1 — **idempotência, tratamento de erro, rastreabilidade e observabilidade**. Hoje esses quatro conceitos deixam de ser boas intenções e viram a engenharia do sistema.

---

## 1. De script a pipeline orquestrado

### O ponto de partida: o que temos depois de cinco encontros

Hoje, o "pipeline" do curso é uma coleção de células de notebook que um humano executa em sequência: extrair `vendas.csv`, limpar com pandas, validar e mandar o refugo para a quarentena, carregar em Parquet, somar o faturamento por categoria. Funciona — **enquanto alguém está presente para clicar, lembrar da ordem e perceber quando algo deu errado**. Isso é um script, não um pipeline.

No Encontro 1 fixamos a distinção: *um script roda uma vez; um pipeline é pensado para rodar muitas vezes, de forma confiável.* A pergunta natural agora é: **o que falta para esse "roda sozinho" existir de verdade?** Quatro problemas concretos aparecem assim que tiramos o humano da cadeira:

- **Dependências.** A transformação só pode rodar depois da extração; a carga, só depois da validação. Quem garante a ordem? No notebook, é a posição das células. Em produção, precisamos declarar essa ordem explicitamente — caso contrário, uma tarefa lê um arquivo que ainda não foi escrito.
- **Agendamento.** O dado chega todo dia às 6h. Alguém precisa disparar o pipeline nesse horário, todos os dias, inclusive feriados e madrugadas. Cron resolve o gatilho; o orquestrador resolve o resto.
- **Falhas.** A fonte veio vazia, a rede caiu no meio da carga, o disco encheu. O que acontece? Sem orquestração, o pipeline para no meio e ninguém fica sabendo até o relatório da diretoria não aparecer. Precisamos de **retry** (tentar de novo, porque muitas falhas são transitórias) e de **alerta** (avisar quando a falha persiste).
- **Reprocessamento.** Descobrimos hoje que o dado de terça veio com um bug na fonte. Precisamos reprocessar **só terça**, sem refazer a semana inteira e sem duplicar o que já está correto. Isso só é seguro se cada tarefa for **idempotente** — exatamente a propriedade que praticamos no E1.

### A virada de mentalidade

Orquestrar é deixar de pensar em "este código que eu rodo" e passar a pensar em **"este sistema que se executa"**. O orquestrador é o componente que conhece o **grafo de tarefas**, sabe a ordem, dispara no horário, repete o que falha, registra tudo o que aconteceu e avisa quando precisa de gente. Em uma frase: **orquestração é dar autonomia confiável ao pipeline.**

> **Pontos de Atenção**
> - Orquestrar cedo demais é over-engineering; orquestrar tarde demais é dívida técnica. O gatilho real é a recorrência: assim que um processo precisa rodar **repetidamente e sem supervisão**, ele merece orquestração.
> - Um agendador puro (cron) dispara, mas não conhece dependências, não tem retry inteligente e não dá visibilidade. Cron é o despertador; o orquestrador é o maestro.
> - A ordem das células de um notebook **não é** uma garantia de produção. O que protege a ordem em produção é a declaração explícita de dependências.

> 🔗 **Conexão com a Prática:** seção *1 — De script a sistema* do notebook mostra o pipeline do curso ainda como funções soltas e expõe, em código, por que a ordem importa.

---

## 2. DAGs — a estrutura que descreve um pipeline

### O que é um DAG

A abstração central da orquestração é o **DAG** — *Directed Acyclic Graph*, ou **grafo dirigido acíclico**. Decompondo o nome:

- **Grafo:** um conjunto de **tarefas** (nós) ligadas por **dependências** (arestas). Cada tarefa é uma unidade de trabalho — extrair, transformar, validar, carregar.
- **Dirigido:** as ligações têm direção. "Transformar depende de extrair" não é o mesmo que o contrário; a seta aponta da causa para o efeito.
- **Acíclico:** **não há ciclos**. Uma tarefa não pode, direta ou indiretamente, depender de si mesma. Se A depende de B e B depende de A, não existe ordem possível de execução — o pipeline travaria. A ausência de ciclos é o que garante que sempre exista uma sequência válida.

O pipeline do curso, desenhado como DAG:

```
extrair → transformar → validar → carregar → sumarizar
```

Linear neste caso, mas DAGs reais ramificam: uma extração pode alimentar três transformações em paralelo, que depois convergem para uma única carga. O grafo captura exatamente quem espera quem.

### Ordem topológica

Dado o grafo de dependências, o orquestrador calcula uma **ordenação topológica**: uma sequência das tarefas em que **toda tarefa aparece depois de todas as suas dependências**. É o algoritmo que responde "em que ordem posso rodar isso sem violar nenhuma dependência?". Quando há ramificações, várias ordens válidas existem — e as tarefas independentes podem rodar em paralelo. No notebook de hoje, **implementamos esse algoritmo do zero**, em Python puro, para desmistificar o que o orquestrador faz por baixo.

### As propriedades que o DAG habilita

- **Agendamento (scheduling).** O DAG tem um gatilho — tipicamente uma expressão **cron** (`0 6 * * *` = "todo dia às 6h"). O scheduler do orquestrador acorda, vê quais DAGs estão na hora, e dispara.
- **Retries.** Cada tarefa pode declarar quantas vezes tentar e com qual intervalo entre tentativas. Falhas transitórias (rede, lock momentâneo) se resolvem sozinhas; só a falha **persistente** vira incidente.
- **Backfill.** Reexecutar o pipeline para um intervalo de datas no passado — útil quando um bug é descoberto ou quando um pipeline novo precisa processar o histórico. O backfill seguro depende, de novo, de **idempotência**: reprocessar terça não pode duplicar nem corromper o que já existe.

> **Pontos de Atenção**
> - O "A" de acíclico não é detalhe acadêmico: um ciclo acidental (uma tarefa que, por engano, depende de um descendente seu) é um bug que **trava** o pipeline. Bons orquestradores detectam e recusam ciclos.
> - Tarefa não é o mesmo que código grande. Tarefas pequenas e bem delimitadas são mais fáceis de repetir, reprocessar e diagnosticar. Granularidade é decisão de engenharia.
> - Agendar não é orquestrar. O cron diz **quando**; o DAG diz **o quê, em que ordem e o que fazer quando falha**.

> 🔗 **Conexão com a Prática:** seção *2 — Um mini-orquestrador de DAG* do notebook define tarefas com dependências em um dicionário, resolve a ordem topológica e executa o pipeline completo do curso.

---

## 3. Apache Airflow — o padrão de mercado

### O que é e por que dominou

O **Apache Airflow** (nascido no Airbnb em 2014, hoje projeto da Apache Software Foundation) popularizou uma ideia poderosa: **definir pipelines como código Python**. Em vez de configurar um pipeline por cliques numa interface, você escreve um arquivo `.py` que **constrói o DAG** — e esse arquivo é versionado no Git, revisado em pull request e testado como qualquer outro código. É a origem da expressão *"data pipelines as code"*.

Os componentes centrais do Airflow:

- **DAG:** o objeto Python que agrupa as tarefas e define o agendamento (`schedule`, `start_date`).
- **Operators:** os tijolos das tarefas. `PythonOperator` roda uma função Python; `BashOperator` roda um comando; há operators específicos para bancos, nuvem, etc. Cada instância de operator é uma **task**.
- **Scheduler:** o processo que acorda, lê os DAGs, decide o que está na hora de rodar e despacha as tarefas para os **workers**.
- **Sensors:** um tipo especial de tarefa que **espera por uma condição** antes de liberar o resto do DAG — por exemplo, "espere o arquivo `vendas.csv` aparecer no diretório". Sensores transformam "rode às 6h" em "rode quando o dado chegar", que é frequentemente mais correto.

### O DAG do curso em Airflow (ilustrativo)

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

with DAG(
    dag_id="pipeline_vendas",
    schedule="0 6 * * *",          # todo dia às 6h
    start_date=datetime(2026, 7, 1),
    catchup=False,
    default_args={"retries": 2},   # cada tarefa tenta 2 vezes extras
) as dag:

    extrair     = PythonOperator(task_id="extrair",     python_callable=tarefa_extrair)
    transformar = PythonOperator(task_id="transformar", python_callable=tarefa_transformar)
    validar     = PythonOperator(task_id="validar",     python_callable=tarefa_validar)
    carregar    = PythonOperator(task_id="carregar",    python_callable=tarefa_carregar)
    sumarizar   = PythonOperator(task_id="sumarizar",   python_callable=tarefa_sumarizar)

    # as dependências viram uma linha legível:
    extrair >> transformar >> validar >> carregar >> sumarizar
```

A última linha é o coração da legibilidade do Airflow: o operador `>>` **declara a aresta do DAG**. Lendo o código, qualquer pessoa entende a ordem. O scheduler do Airflow lê esse arquivo, monta o grafo, e a cada dia às 6h executa as tarefas na ordem topológica, aplicando os retries definidos.

> **Pontos de Atenção**
> - O ponto forte do Airflow é a maturidade: imenso ecossistema de operators, comunidade enorme, interface web para monitorar execuções. É o **padrão seguro** para times grandes.
> - O ponto fraco histórico: o Airflow é **centrado em tarefas**, não em dados. Ele sabe que a tarefa rodou, mas não tem, nativamente, a noção do **dado que ela produziu**. É a lacuna que os orquestradores modernos atacam (seção 4).
> - `catchup=False` é uma trava de segurança importante: sem ela, ao subir um DAG com `start_date` no passado, o Airflow tenta executar **todas** as datas perdidas de uma vez.

> 🔗 **Conexão com a Prática:** seção *3 — O mesmo DAG em Airflow* do notebook exibe (sem executar) este código e o compara com o mini-orquestrador da seção 2.

---

## 4. Orquestração moderna asset-based — Dagster e Prefect

### A mudança de eixo: da tarefa para o dado

A geração mais recente de orquestradores parte de uma observação: **o que importa em um pipeline de dados não é "a tarefa rodou", e sim "o dado existe, está atualizado e está correto".** Essa inversão de foco — de *tarefas* para *ativos de dados* (data assets) — é a tendência mais marcante da área e organiza as duas ferramentas a seguir.

### Dagster — assets / software-defined assets

O **Dagster** propõe que você declare **os ativos** que o pipeline produz (a tabela limpa, o Parquet, o sumário), e não as tarefas. Cada ativo é uma função decorada que declara **de quais outros ativos depende**. O orquestrador deriva o grafo a partir dessas dependências de dados — é a abordagem **declarativa / asset-based**.

```python
from dagster import asset

@asset
def vendas_brutas():
    return ler_csv("vendas.csv")

@asset
def vendas_limpas(vendas_brutas):     # depende do ativo acima — a aresta é o argumento
    return transformar(vendas_brutas)

@asset
def vendas_parquet(vendas_limpas):
    return carregar_parquet(vendas_limpas)
```

Repare: a dependência é o **argumento da função**. Você descreve *o quê* cada dado é e *de onde* ele vem; o Dagster monta o DAG, sabe a linhagem (lineage) de cada ativo, e oferece de fábrica testes, materializações parciais e integração com observabilidade. É a materialização concreta de *"o dado como produto"* que defendemos desde o E1.

### Prefect — flows e tasks, com dinamismo

O **Prefect** mantém o vocabulário de **tasks** e **flows**, mas com a ergonomia de Python moderno: você decora funções com `@task` e `@flow` e **chama-as como funções normais** — o Prefect rastreia as dependências a partir das chamadas. Seu diferencial é o **dinamismo**: o grafo pode ser construído em tempo de execução (ex.: uma tarefa por arquivo encontrado, número decidido só na hora).

```python
from prefect import flow, task

@task(retries=2)
def extrair(): ...

@task
def transformar(dados): ...

@flow
def pipeline_vendas():
    dados = extrair()
    limpos = transformar(dados)     # a dependência nasce do uso do resultado
    return limpos
```

### Quando preferir o Airflow

A tendência aponta para asset-based, mas a escolha é de engenharia, não de moda:

- **Prefira Airflow** quando o time já o domina, quando há grande variedade de integrações prontas a explorar, ou quando o problema é genuinamente **orientado a tarefas** (orquestrar jobs heterogêneos que não são só "produzir dados").
- **Prefira Dagster** quando o foco é **produzir e versionar ativos de dados** com linhagem e testes de primeira classe — pipelines analíticos modernos.
- **Prefira Prefect** quando você precisa de **fluxos dinâmicos** e de uma curva de adoção suave a partir de código Python existente.

> **Pontos de Atenção**
> - Asset-based não é "melhor" em abstrato; é melhor quando a pergunta operacional é *"este dado está fresco e correto?"* em vez de *"este job terminou?"*.
> - Migrar de orquestrador é caro. A decisão importa mais pela aderência ao **modelo mental do time** do que por benchmarks.
> - Todos os três rodam DAGs e fazem retries, logging e agendamento. A diferença é o **eixo conceitual** (tarefa vs. ativo) e a ergonomia.

> 🔗 **Conexão com a Prática:** seção *4 — Prefect e Dagster* do notebook mostra o mesmo pipeline do curso nas duas ferramentas (código ilustrativo, não executado) e discute as diferenças.

---

## 5. Produção — o que separa "rodou" de "confiável"

Colocar o DAG no ar é metade do caminho. A outra metade é o conjunto de práticas que tornam o pipeline **confiável ao longo do tempo**, sem alguém de plantão olhando logs.

### Observabilidade operacional

- **Logging.** Cada tarefa registra início, fim, status e contagens (linhas lidas, descartadas, quarentenadas). É a rastreabilidade do E1 e a quarentena do E5 chegando à camada de execução: você consulta *o que* o pipeline fez sem assistir rodar.
- **Alerting.** Quando uma tarefa esgota seus retries, o orquestrador **avisa** — e-mail, Slack, PagerDuty. A regra de ouro: **alerte sobre o que exige ação humana** e silencie o ruído, ou o time aprende a ignorar os alertas.
- **Retries.** Política explícita por tarefa: número de tentativas e intervalo (idealmente com *backoff* — esperar mais a cada falha). Absorve o transitório; preserva o atendimento humano para o persistente.
- **SLAs.** Um contrato de tempo: "o dado de vendas estará pronto até 7h". Se o DAG não cumpre o SLA, isso é, por si só, um alerta — mesmo que nenhuma tarefa tenha "falhado". Atraso é uma forma de falha.
- **Monitoramento.** A visão histórica: taxa de sucesso por dia, duração das execuções, tarefas que mais falham. É onde a orquestração **converge com observabilidade** — a fronteira entre "o pipeline rodou" e "o dado está saudável" se dissolve.

### CI/CD para dados

Como o pipeline **é código** (a lição do Airflow), ele entra no mesmo rigor de engenharia de software:

- **Versionar o pipeline:** o DAG vive no Git. Toda mudança é um commit, revisada em pull request, com histórico e possibilidade de reverter.
- **Testar antes de subir:** testes de unidade nas funções de transformação (a limpeza faz o que promete?), testes nas regras de validação (a quarentena pega o refugo certo?), e checagem de que o DAG não tem ciclos e que importa sem erros.
- **Promover por ambientes:** o pipeline sobe primeiro em *staging* (com dado de teste), e só depois em *produção*. Nunca se descobre um pipeline quebrado em produção quando o pipeline de entrega o pegaria antes.

> **Pontos de Atenção**
> - Pipeline em produção sem alerta é uma bomba-relógio silenciosa: você só descobre a falha pelo consumidor reclamando — exatamente o que a observabilidade existe para evitar.
> - Retry infinito mascara problema crônico. Limite as tentativas; transforme a falha persistente em incidente visível.
> - "Testar dado" e "testar código de dados" são coisas diferentes e ambas necessárias: o E5 testou o **dado** (qualidade/contratos); o CI/CD testa o **código** que processa o dado.

> 🔗 **Conexão com a Prática:** seção *5 — Produção* do notebook adiciona logging estruturado com timestamps e status por tarefa, e demonstra retries recuperando uma falha simulada.

---

## 6. Fechamento da disciplina — o pipeline E1→E6 como um sistema único

### Os seis encontros eram um só pipeline

Olhe para trás e veja que **nunca trocamos de assunto**. O mesmo `vendas.csv` sintético atravessou a disciplina inteira, ganhando uma camada de engenharia por encontro:

| Encontro | O que adicionamos ao pipeline |
|---|---|
| **E1 — Fundamentos** | O vocabulário e o primeiro ETL em Python puro: extrair → transformar → carregar, com rastreabilidade e idempotência plantadas. |
| **E2 — pandas** | Substituímos laços manuais por operações de tabela; a transformação ficou expressiva e concisa. |
| **E3 — Transformação e SQL** | Aprendemos a empurrar trabalho para o banco e a pensar em conjuntos; a mesma pergunta, respondida de duas formas. |
| **E4 — Escala** | Quebramos a premissa "cabe na memória"; processamento em lote, particionamento, o problema do volume. |
| **E5 — Qualidade** | Demos voz ao dado ruim: validação, contratos e **quarentena** — o refugo deixou de poluir o resultado. |
| **E6 — Orquestração** | Tiramos o humano da cadeira: o conjunto de scripts virou **um DAG que roda sozinho, em ordem, com retry, log e alerta**. |

O pipeline de hoje — `extrair → transformar → validar → carregar → sumarizar` — **não é um exemplo novo**: é a soma de tudo, agora autônoma. A idempotência do E1 é o que torna o backfill seguro; a rastreabilidade do E1 é o que vira logging em produção; a quarentena do E5 é o que o alerting monitora. **Os princípios não mudaram — ganharam um corpo que se executa.**

### O mapa do que você aprendeu

Você sai desta disciplina sabendo: ler e escrever dados em Python; manipular tabelas com pandas; transformar e consultar com SQL; raciocinar sobre escala; garantir qualidade com validação e contratos; e orquestrar tudo isso como um sistema de produção. Mais importante que as ferramentas: você sai com a **mentalidade** de tratar dado como produto e pipeline como sistema — confiável, rastreável, testável.

### Próximos passos na carreira de Engenharia de Dados

- **Aprofunde SQL e modelagem de dados** — é a habilidade de maior retorno e a mais duradoura da área.
- **Construa um projeto de portfólio de ponta a ponta** — pegue uma fonte pública, monte o pipeline E1→E6 completo, versione no Git, agende num orquestrador. Vale mais que qualquer certificado.
- **Suba à nuvem** — os conceitos são os mesmos; o que muda é o ambiente (armazenamento de objetos, data warehouses gerenciados, orquestradores em nuvem).
- **Estude sistemas distribuídos** — a leitura de Kleppmann é o passo natural para entender *por que* as ferramentas de dados são como são.
- **Pratique a disciplina de produção** — testes, CI/CD, observabilidade. É o que separa quem escreve um pipeline de quem mantém um sistema.

> **Caixa de tendência — orquestração declarativa e a convergência com observabilidade**
> O movimento de fundo da orquestração moderna é **declarativo / asset-based**: você declara *quais ativos de dados devem existir e como se derivam* (Dagster), e o orquestrador deduz o grafo, a linhagem e quando rematerializar. É a maturação do lema **"data orchestration as code"** — o pipeline deixa de ser uma lista de passos a executar e passa a ser uma **descrição do estado desejado dos dados**. Em paralelo, a fronteira entre orquestração e **observabilidade** se dissolve: saber *que a tarefa rodou* e saber *que o dado está fresco e correto* viram a mesma pergunta. O futuro próximo da função é menos "agendar jobs" e mais "garantir, de forma declarativa e observável, que os dados certos existem na hora certa".

> **Pontos de Atenção**
> - Ferramentas mudam; princípios permanecem. Quem aprende **só a ferramenta** envelhece com ela; quem aprende **idempotência, contrato, rastreabilidade e orquestração** atravessa qualquer stack.
> - O melhor portfólio é um pipeline que **roda sozinho e se explica**. Demonstre os seis encontros num único repositório.

> 🔗 **Conexão com a Prática:** seção *Encerramento do curso* do notebook executa o pipeline E1→E6 completo, orquestrado, e fecha com o mapa da disciplina.

---

## Síntese do encontro

1. **De script a pipeline:** tirar o humano da cadeira expõe quatro problemas — dependências, agendamento, falhas e reprocessamento — que a orquestração resolve.
2. **DAG** é a estrutura central: tarefas (nós) e dependências (arestas), dirigido e **acíclico**, executado em ordem topológica, com agendamento (cron), retries e backfill.
3. **Apache Airflow** é o padrão de mercado: pipelines como código Python, com operators, scheduler e sensors — maduro, porém centrado em tarefas.
4. **Dagster (assets) e Prefect (flows dinâmicos)** representam a virada **asset-based**, focando no dado produzido e não na tarefa; a escolha é de modelo mental, não de moda.
5. **Produção** exige logging, alerting, retries, SLAs, monitoramento e **CI/CD para dados** — porque o pipeline é código e merece o mesmo rigor.
6. **Os seis encontros eram um só pipeline:** os princípios do E1 (idempotência, rastreabilidade) ganharam corpo na orquestração do E6.

### Encerramento do curso

A disciplina termina, mas o pipeline não. Você começou no Encontro 1 com um CSV sujo e 20 linhas de Python; sai hoje sabendo orquestrar um sistema de dados de produção. O fio condutor — **extrair, transformar, carregar, com confiabilidade** — nunca mudou; o que cresceu foi a sua capacidade de sustentá-lo em escala, com qualidade e autonomia.

Leve uma ideia acima de todas: **um script roda uma vez; um pipeline roda sempre, sozinho e confiável.** Construir esse "sempre" é a Engenharia de Dados. Bons pipelines — e até a próxima.

---

## Bibliografia

- REIS, Joe; HOUSLEY, Matt. *Fundamentos de Engenharia de Dados*. O'Reilly / Alta Books, 2023.
- MCKINNEY, Wes. *Python para Análise de Dados*. 3ª ed. O'Reilly, 2022.
- KLEPPMANN, Martin. *Designing Data-Intensive Applications*. O'Reilly, 2017.

---

*Material de apoio — UNIFOR Pós-Graduação · Prof. Cassio Pinheiro. Encerra a disciplina; acompanha o notebook prático e os slides do Encontro 6.*
