# Linguagens de Programação para Engenharia de Dados
## Encontro 4 — Processamento de dados em escala

**Instituição:** Universidade de Fortaleza (UNIFOR)
**Curso:** Pós-Graduação Lato Sensu — Especialização em Engenharia de Dados
**Professor:** Cassio Pinheiro · [LinkedIn](https://www.linkedin.com/in/cassio-pinheiro-9baa7939/)
**Data:** 09/07/2026 · 19:00 às 22:30
**Carga horária da disciplina:** 24h

---

## Como ler este documento

Este é o material de fundamentação do **quarto encontro**. Ele acompanha:

- o **notebook** `e4_pratica.ipynb` (onde rodamos código — cada seção do notebook é referenciada aqui com a marca 🔗);
- os **slides** `e4_slides.pptx` (para a aula expositiva).

Nos encontros anteriores construímos o vocabulário (E1), domamos o pandas e a manipulação de tabelas (E2) e exercitamos transformações e SQL (E3). Tudo isso assumiu, sem dizer, uma premissa confortável: **o dado cabe na memória**. Hoje quebramos essa premissa de propósito. Vamos pegar o nosso dataset de vendas — que até agora tinha dezenas de linhas — e **escalá-lo para mais de 200 mil transações** (sintéticas e determinísticas) para *sentir* o problema da escala antes de falar das soluções.

A pergunta que organiza o encontro é simples e brutal: **o que muda quando o dado é grande demais?** Muda tudo — o formato em que você grava, como você lê, quando você executa o cálculo e em qual máquina (ou máquinas) ele roda. Vamos passar dos formatos colunares (Parquet/Arrow) ao processamento em lote, à avaliação tardia (Polars *lazy*), ao processamento distribuído (PySpark) e, por fim, à decisão de engenharia que mais importa: **qual ferramenta usar para qual tamanho de problema**.

> **Combinado pedagógico:** escala não é "rodar o mesmo código numa máquina maior". É escolher a abstração certa. O melhor engenheiro de dados de 2026 muitas vezes é aquele que percebe que **não precisava de um cluster** — bastava ler menos dado.

---

## 1. O problema da escala: quando o dado não cabe na memória

### O ponto de virada

Todo o código que escrevemos até aqui carregava o arquivo inteiro para a RAM com um `pd.read_csv(...)`. Isso funciona enquanto o arquivo for menor que a memória disponível — e, na prática, bem menor, porque uma cópia de um CSV de 1 GB pode ocupar 3 a 10 GB depois de carregado em um DataFrame (tipos, índices, *strings* em Python têm sobrecusto). Quando o dado ultrapassa esse limite, o processo não fica "lento": ele **morre** com um `MemoryError`, ou o sistema operacional começa a usar disco como memória (*swap*) e a execução despenca para uma lentidão inviável.

Esse é o **ponto de virada** da Engenharia de Dados em escala: o momento em que a estratégia "carregue tudo e processe" deixa de ser uma opção. A partir daí, todas as ferramentas que veremos hoje existem para responder à mesma necessidade — **processar dados maiores que a memória da máquina** — por caminhos diferentes.

### Duas métricas que você precisa separar: throughput vs. latência

Engenheiros iniciantes falam de "rápido" como se fosse uma coisa só. Não é. Há duas grandezas distintas:

- **Latência** — quanto tempo leva *uma* operação individual. "Quanto tempo para responder a esta consulta?" Importa para sistemas interativos (um dashboard, uma API).
- **Throughput** (vazão) — quantos itens você processa *por unidade de tempo*. "Quantas transações por segundo o pipeline ingere?" Importa para processamento em lote de grandes volumes.

São métricas que frequentemente competem. Um sistema otimizado para latência mínima (responder rápido a uma requisição) pode ter throughput menor; um sistema otimizado para throughput máximo (processar bilhões de linhas à noite) pode ter latência alta em cada item individual. Kleppmann insiste neste ponto: ao projetar um sistema *data-intensive*, você precisa saber **qual das duas você está otimizando**, porque a resposta muda toda a arquitetura.

### Escalar para cima vs. escalar para os lados

Quando o dado cresce, há dois caminhos para dar conta dele:

- **Escala vertical (*scale up*)** — usar uma máquina maior: mais RAM, mais CPU, discos mais rápidos. É simples (nada muda no código), mas tem teto físico e o custo cresce de forma não-linear (uma máquina com 1 TB de RAM custa muito mais que dez máquinas de 100 GB).
- **Escala horizontal (*scale out*)** — distribuir o trabalho por **várias máquinas** que processam pedaços do dado em paralelo. Não tem teto prático, mas introduz uma complexidade enorme: coordenação, rede, falhas parciais, dados que precisam viajar entre máquinas.

A grande virada da última década é que **a escala vertical ficou muito mais poderosa e barata**. Uma máquina na nuvem com centenas de GB de RAM é trivial de alugar por hora. Isso reabilitou ferramentas *single-node* potentes (Polars, DuckDB) que, em uma máquina robusta, dão conta de volumes que antes "exigiam" um cluster. Voltaremos a isso na Seção 6.

> **Pontos de Atenção**
> - "Memória cheia" não é "lento" — é **falha**. Pipelines em escala precisam de uma estratégia explícita para dados maiores que a RAM, não de torcer para caber.
> - Antes de subir para um cluster, pergunte: **será que cabe numa máquina grande?** Distribuição tem um custo de complexidade que frequentemente não compensa.
> - Latência e throughput são objetivos diferentes e às vezes opostos. Saiba qual você está otimizando.

> 🔗 **Conexão com a Prática:** seção *1 — Sentindo a escala* do notebook gera 200 mil+ transações sintéticas determinísticas e mostra, em números, a memória que esse dado ocupa.

---

## 2. Formatos colunares: por que Parquet vence em analytics

### Linha vs. coluna: a mesma tabela, dois arranjos no disco

Um CSV guarda os dados **por linha**: a primeira transação inteira, depois a segunda inteira, e assim por diante. É natural para humanos e para sistemas que leem ou gravam um registro de cada vez (um banco transacional). Mas é péssimo para análise.

A maioria das perguntas analíticas toca **poucas colunas de muitas linhas**: "qual o faturamento total por categoria?" precisa apenas de `categoria` e `valor` — não importa o `cliente`, a `cidade`, o `id`. Num CSV (formato por linha), para ler essas duas colunas o computador precisa **varrer o arquivo inteiro**, atravessando todas as colunas de todas as linhas. Desperdício puro.

Um **formato colunar** guarda os dados **por coluna**: todos os valores de `categoria` juntos, depois todos os valores de `valor` juntos. Isso desbloqueia ganhos enormes em análise.

### Por que colunar ganha: três mecanismos

- **Leitura por colunas (*column pruning* / *projection pushdown*)** — se a consulta só pede `categoria` e `valor`, o motor lê **apenas esses blocos** do disco e ignora o resto. Numa tabela com 30 colunas onde você usa 2, isso é uma economia de ~93% de I/O.
- **Compressão muito mais eficiente** — valores da *mesma coluna* são do mesmo tipo e frequentemente repetitivos (a coluna `categoria` tem só 6 valores distintos). Algoritmos de compressão e codificações como *dictionary encoding* e *run-length encoding* funcionam muito melhor sobre dados homogêneos. Um Parquet costuma ocupar de **3 a 10 vezes menos espaço** que o CSV equivalente.
- **Filtro por estatísticas (*predicate pushdown*)** — o Parquet guarda metadados (mínimo, máximo, contagem) por blocos de linhas (*row groups*). Se você filtra `data >= '2026-06-01'` e o bloco inteiro tem máximo `2026-05-30`, o motor **pula o bloco sem nem ler**.

### Parquet e Apache Arrow: formato em disco e formato em memória

Vale distinguir dois nomes que aparecem juntos:

- **Apache Parquet** é um formato de **arquivo em disco** — colunar, comprimido, com metadados e estatísticas. É o padrão de fato para armazenar dados analíticos.
- **Apache Arrow** é um padrão de **representação colunar em memória** — uma especificação que define exatamente como uma tabela colunar é organizada na RAM. Sua importância é a **interoperabilidade**: pandas, Polars, DuckDB, Spark e várias outras ferramentas podem trocar dados *sem serializar e desserializar*, porque todas falam Arrow. É o "idioma comum" que conecta o ecossistema moderno.

> **Pontos de Atenção**
> - CSV é ótimo para **intercâmbio com humanos e sistemas legados**; é ruim como formato de armazenamento analítico. Para dados que você vai reprocessar muitas vezes, **grave em Parquet**.
> - O ganho do Parquet não é só tamanho de arquivo — é **menos I/O e menos CPU** na leitura, exatamente onde pipelines em escala gastam tempo.
> - Arrow não é "mais uma biblioteca": é o padrão que permite trocar dados entre ferramentas sem cópia. Quando duas ferramentas "falam Arrow", integrá-las é barato.

> 🔗 **Conexão com a Prática:** seção *2 — CSV vs. Parquet* do notebook grava o mesmo dataset nos dois formatos e **mede, em disco e em tempo de leitura**, a diferença.

---

## 3. Particionamento e processamento em lote

### A ideia: não carregue tudo de uma vez

Se o dado não cabe na memória, a primeira estratégia (e a mais antiga) é processá-lo **em pedaços** (*chunks*): leia um bloco de N linhas, processe, descarte da memória, leia o próximo bloco, e vá **acumulando o resultado**. A memória usada fica limitada ao tamanho de um *chunk*, não ao do arquivo inteiro — então você processa um arquivo de 100 GB numa máquina de 8 GB de RAM.

O pandas oferece isso diretamente com o parâmetro `chunksize`:

```python
faturamento = {}
for bloco in pd.read_csv("vendas_grande.csv", chunksize=50_000):
    parcial = bloco.groupby("categoria")["valor"].sum()
    for categoria, valor in parcial.items():
        faturamento[categoria] = faturamento.get(categoria, 0) + valor
```

Repare no padrão: cada *chunk* produz um resultado **parcial**, e nós o **combinamos** com o acumulado. Isso só funciona para agregações que podem ser calculadas incrementalmente — somas, contagens, mínimos, máximos. (Uma mediana exata, por exemplo, é mais difícil, porque precisa ver todos os valores juntos.) Esse padrão "processar pedaço, combinar parcial" é a **semente conceitual** do que o Spark fará automaticamente entre máquinas na Seção 5.

### Particionamento: organizar o dado por chaves de acesso

Uma segunda estratégia, complementar, é **particionar** o dado fisicamente. Em vez de um arquivo gigante `vendas.parquet`, você grava uma estrutura de pastas:

```
vendas/
  data=2026-07-01/parte.parquet
  data=2026-07-02/parte.parquet
  data=2026-07-03/parte.parquet
```

Agora, se a consulta filtra `data = '2026-07-02'`, o motor lê **apenas aquela pasta** e ignora completamente as outras — isso é *partition pruning*, o predicate pushdown elevado ao nível do sistema de arquivos. Particiona-se tipicamente por colunas usadas em filtros frequentes: **data** (a mais comum) e **categoria** são exemplos clássicos.

> **Pontos de Atenção**
> - Chunking troca **memória por tempo**: você usa pouca RAM, mas faz mais idas ao disco. É a solução certa quando a máquina é pequena e o dado é grande.
> - Nem toda agregação é incremental. Soma, contagem, mínimo e máximo somam-se entre blocos; mediana e contagem de distintos exatos exigem cuidado extra.
> - Particionar bem é uma decisão de modelagem: particione pelas colunas que aparecem nos **filtros** dos consumidores. Particionar errado (ex.: por uma coluna de alta cardinalidade como `id`) cria milhões de arquivos minúsculos e piora tudo.

> 🔗 **Conexão com a Prática:** seção *3 — Processamento em lote* do notebook soma o faturamento por categoria **em chunks**, sem nunca carregar o arquivo inteiro na memória.

---

## 4. Avaliação tardia (lazy): deixe o motor pensar antes de executar

### Eager vs. lazy

O pandas é **eager** (ávido): cada operação executa na hora em que você a escreve. Você lê o arquivo (executa agora), filtra (executa agora, materializando um DataFrame intermediário), seleciona colunas (executa agora), agrega (executa agora). O resultado é correto, mas o motor **nunca viu o plano completo** — ele executou cada passo isoladamente, sem chance de otimizar o conjunto.

A **avaliação tardia (*lazy*)** inverte isso. Em vez de executar imediatamente, cada operação apenas **registra a intenção**, construindo um **plano de consulta** (uma descrição do que você quer). Nada acontece de fato até você pedir o resultado — em Polars, chamando `.collect()`. Nesse momento, o motor **olha o plano inteiro de uma vez** e o **otimiza** antes de executar.

### O que a otimização ganha

Considere: "ler o Parquet, filtrar `categoria == 'eletronicos'`, e somar o `valor`". Um motor *lazy* percebe, ao olhar o plano completo, que:

- só precisa das colunas `categoria` e `valor` → faz **projection pushdown** (lê só essas colunas do Parquet);
- o filtro pode ser aplicado **na leitura** → faz **predicate pushdown** (descarta linhas antes de carregá-las);
- pode pular *row groups* inteiros usando as estatísticas do Parquet → lê ainda menos do disco.

O resultado é que o motor *lazy* frequentemente lê uma **fração** do dado que um pipeline *eager* leria. É exatamente o princípio que dá título à nossa citação de hoje: **não processe o que você não precisa ler.**

### Polars: lazy de verdade, em Rust

**Polars** é uma biblioteca de DataFrames escrita em **Rust**, projetada de origem para escala e paralelismo. Seu modo *lazy* começa não com `read_parquet` (que carregaria tudo), mas com `scan_parquet`, que apenas registra a fonte sem ler nada:

```python
import polars as pl

resultado = (
    pl.scan_parquet("vendas_grande.parquet")   # não lê nada ainda
      .filter(pl.col("categoria") == "eletronicos")
      .group_by("categoria")
      .agg(pl.col("valor").sum())
      .collect()                               # AGORA executa o plano otimizado
)
```

Além da otimização de plano, o Polars **usa todos os núcleos da CPU** automaticamente e opera sobre dados em formato Arrow. Para o tipo de carga que estamos vendo hoje — agregações sobre milhões de linhas em uma única máquina — ele costuma ser **ordens de magnitude mais rápido** que o pandas, e ainda lida com dados maiores que a memória no modo *streaming*.

> **Pontos de Atenção**
> - Lazy não é "mais devagar para começar": é **mais inteligente**. O atraso na execução é o que permite a otimização global.
> - O par `scan_parquet` + `collect()` é o idioma central do Polars *lazy*. Use `scan_*` (não `read_*`) sempre que quiser que os filtros sejam empurrados para a leitura.
> - O ganho do *lazy* é maior justamente onde mais importa: quando você só precisa de **um recorte pequeno** de um dado **grande**.

> 🔗 **Conexão com a Prática:** seção *4 — Polars lazy* do notebook roda a mesma agregação via `scan_parquet`/`collect()` e compara o tempo com a versão pandas.

---

## 5. Processamento distribuído: quando uma máquina não basta

### A ideia do Spark

Quando nem a maior máquina disponível dá conta — pense em petabytes — é preciso **distribuir** o trabalho por um **cluster** de máquinas. O **Apache Spark** é o motor distribuído mais consagrado. A abstração que ele oferece é um **DataFrame distribuído**: para o programador, parece uma tabela única; por baixo, está **fatiada em partições espalhadas por dezenas ou centenas de máquinas**, cada uma processando seu pedaço em paralelo.

### Transformações vs. ações: o Spark também é lazy

A peça conceitual mais importante do Spark — e a razão de o termos colocado *depois* do Polars — é que ele opera com **avaliação tardia**, exatamente como acabamos de ver. As operações se dividem em duas categorias:

- **Transformações** (`filter`, `select`, `groupBy`, `join`, `withColumn`) — descrevem *o que* você quer, mas **não executam nada**. Apenas adicionam um nó ao plano. São *lazy*.
- **Ações** (`count`, `collect`, `show`, `write`) — pedem um **resultado concreto**. É só aqui que o Spark dispara a computação.

Quando uma ação é chamada, o Spark monta um **DAG** (grafo acíclico dirigido) de todas as transformações pendentes, **otimiza** esse grafo (com um otimizador chamado Catalyst, do mesmo espírito do que vimos em Polars) e só então distribui o trabalho pelo cluster. Em PySpark:

```python
from pyspark.sql import SparkSession, functions as F

spark = SparkSession.builder.appName("vendas").getOrCreate()

df = spark.read.parquet("vendas_grande.parquet")   # transformação (lazy)
resultado = (
    df.filter(F.col("categoria") == "eletronicos")  # transformação (lazy)
      .groupBy("categoria")                          # transformação (lazy)
      .agg(F.sum("valor").alias("faturamento"))      # transformação (lazy)
)
resultado.show()                                     # AÇÃO — dispara o cluster
```

### Particionamento e shuffle: o custo escondido

Operações como `groupBy` e `join` precisam reunir, numa mesma máquina, todas as linhas que compartilham a mesma chave (todos os `eletronicos` juntos para somar). Mas essas linhas estão **espalhadas** pelas partições do cluster. O Spark então move dados entre as máquinas pela rede — o famoso **shuffle**. O shuffle é a operação **mais cara** de um job distribuído, porque envolve serialização e tráfego de rede. Boa parte da otimização de Spark consiste em **minimizar shuffle**. Esse é o preço da escala horizontal: a coordenação entre máquinas, ausente quando tudo cabe numa só, passa a dominar o custo.

> **Pontos de Atenção**
> - Spark é poderoso e **pesado**: subir um cluster, configurar, distribuir o job tem um custo fixo alto. Para dados que cabem em uma máquina, esse overhead frequentemente **não compensa**.
> - O mesmo conceito de *lazy* + *plano otimizado* aparece em Polars e em Spark. Aprender um facilita muito o outro.
> - `shuffle` é onde os jobs distribuídos engasgam. Quando um job Spark está lento, o suspeito número um é shuffle excessivo.

> 🔗 **Conexão com a Prática:** seção *5 — PySpark (ilustrativo)* do notebook traz o código em PySpark dentro de um `try/except`: se o Spark não estiver instalado (não vamos instalá-lo), a célula **imprime o código equivalente e explica transformações vs. ações** — sem erro.

---

## 6. Qual ferramenta usar: o mapa de decisão

### A pergunta certa não é "qual a melhor", é "qual o tamanho do problema"

Não existe ferramenta universalmente melhor. Existe a **ferramenta certa para o tamanho do dado e a forma do trabalho**. Um mapa prático para 2026:

| Situação | Ferramenta recomendada | Por quê |
|---|---|---|
| Cabe folgado na RAM (até alguns GB) | **pandas** | Ecossistema, familiaridade, "cola" com tudo |
| Grande para uma máquina, mas cabe num servidor potente (dezenas a centenas de GB) | **Polars / DuckDB** | *Single-node*, lazy, multi-core, sem overhead de cluster |
| Grande demais para qualquer máquina (TB a PB) | **Spark** (ou similar distribuído) | Escala horizontal real, tolerância a falhas |

### "Small-data engines": o ressurgimento do single-node

Durante os anos 2010, a resposta para "dado grande" era quase automaticamente "use Spark". Mas duas coisas mudaram: as máquinas ficaram **muito maiores e mais baratas**, e surgiram motores *single-node* extremamente eficientes — **DuckDB** (um banco analítico colunar embutido, como "o SQLite do analytics") e **Polars**. Hoje, a maioria dos datasets reais — que raramente passam de algumas dezenas de GB — roda **mais rápido e mais barato** numa única máquina com DuckDB ou Polars do que num cluster Spark, sem a complexidade operacional. A lição de engenharia: **não pague o custo de um cluster por um problema que cabe num laptop robusto.**

### Lakehouse: o fim da separação data lake + data warehouse

Historicamente havia dois mundos separados: o **data lake** (armazenamento barato de arquivos brutos, ex.: Parquet em object storage, flexível mas sem garantias) e o **data warehouse** (banco analítico estruturado, com transações e governança, mas caro e rígido). Você copiava dados de um para o outro, mantendo duas cópias e dois sistemas.

O **Lakehouse** unifica os dois: dá ao data lake as garantias do warehouse. A peça-chave são os **formatos de tabela abertos** — **Delta Lake** e **Apache Iceberg** — que adicionam, por cima de arquivos Parquet em object storage, uma camada de metadados que entrega **transações ACID, versionamento (*time travel*), evolução de esquema e atualizações/deleções**. Você mantém **uma única cópia** dos dados, em formato aberto, e ferramentas diferentes (Spark, Polars, DuckDB, engines de SQL) leem a mesma tabela. É a arquitetura de dados dominante de 2026.

> **Caixa de tendência — para onde a área vai**
> - **A ascensão de Polars e DuckDB.** O *single-node* potente reabilitou o "small data" (que, na prática, é a maioria dos casos). Muitos pipelines que rodavam em Spark estão migrando para Polars/DuckDB — mais simples, mais barato, mais rápido.
> - **Arrow como padrão de interoperabilidade.** O formato colunar em memória virou o "idioma comum": ferramentas trocam dados sem custo de conversão. Isso reduz o atrito de combinar tecnologias.
> - **O Lakehouse substituindo o "lake + warehouse" separado.** Delta Lake e Iceberg trazem garantias de banco para arquivos abertos. A separação histórica entre lago e armazém está desaparecendo numa arquitetura única, aberta e governada.

> **Pontos de Atenção**
> - Comece pela menor ferramenta que resolve. Suba a complexidade (single-node → distribuído) **só quando o tamanho exigir**, medindo, não por reflexo.
> - "Formato aberto" (Parquet, Iceberg, Delta) protege você do aprisionamento a um fornecedor. É uma decisão estratégica, não só técnica.
> - A ferramenta certa muda com o tempo. O que não muda são os **princípios**: colunar, lazy, particionar, ler menos.

> 🔗 **Conexão com a Prática:** seção *6 — Qual ferramenta* do notebook consolida o mapa de decisão e fecha a comparação de tempos entre as abordagens vistas.

---

## Síntese do encontro

Hoje aprendemos a pensar **em escala**:

1. **O problema da escala** aparece quando o dado não cabe na memória — e a partir daí precisamos de estratégias explícitas, distinguindo **latência** de **throughput** e **escala vertical** de **horizontal**.
2. **Formatos colunares** (Parquet em disco, Arrow em memória) vencem em análise por lerem só as colunas necessárias, comprimirem melhor e pularem blocos via estatísticas (*pushdown*).
3. **Processamento em lote** (*chunking*) e **particionamento** permitem processar dados maiores que a RAM, somando resultados parciais e lendo só as partições relevantes.
4. **Avaliação tardia (*lazy*)** — com Polars `scan_parquet`/`collect()` — deixa o motor otimizar o plano inteiro antes de executar, lendo uma fração do dado.
5. **Processamento distribuído** com Spark escala para os lados: DataFrame distribuído, transformações *lazy* vs. ações, DAG otimizado e o custo do **shuffle**.
6. **Qual ferramenta usar** depende do tamanho: pandas -> Polars/DuckDB -> Spark; e o **Lakehouse** (Delta/Iceberg) unifica lago e armazém em formatos abertos.

### Para o próximo encontro

- Rode o notebook por conta própria; **aumente o número de linhas** (de 200 mil para 1 milhão) e observe como cada abordagem reage — onde o pandas sofre e onde Polars/Parquet brilham.
- No **Encontro 5**, voltamos a olhar o dado de perto: **qualidade de dados** — validação, contratos, testes e como garantir que o dado em escala não seja apenas grande, mas **confiável**. Traga em mente: de que adianta processar 200 milhões de linhas rápido se uma fração delas está errada?

---

## Bibliografia

- KLEPPMANN, Martin. *Designing Data-Intensive Applications*. O'Reilly, 2017.
- REIS, Joe; HOUSLEY, Matt. *Fundamentos de Engenharia de Dados*. O'Reilly / Alta Books, 2023.
- MCKINNEY, Wes. *Python para Análise de Dados*. 3ª ed. O'Reilly, 2022.

---

*Material de apoio — UNIFOR Pós-Graduação · Prof. Cassio Pinheiro. Acompanha o notebook prático e os slides da disciplina.*
