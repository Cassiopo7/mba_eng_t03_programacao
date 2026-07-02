# Linguagens de Programação para Engenharia de Dados
## Encontro 3 — SQL para Engenharia de Dados

**Instituição:** Universidade de Fortaleza (UNIFOR)
**Curso:** Pós-Graduação Lato Sensu — Especialização em Engenharia de Dados
**Professor:** Cassio Pinheiro · [LinkedIn](https://www.linkedin.com/in/cassio-pinheiro-9baa7939/)
**Data:** 27/06/2026 · 19:00 às 22:30
**Carga horária da disciplina:** 24h

---

## Como ler este documento

Este é o material de fundamentação do **terceiro encontro**. Ele acompanha:

- o **notebook** `e3_pratica.ipynb` (onde rodamos SQL de verdade — cada seção do notebook é referenciada aqui com a marca 🔗);
- os **slides** `e3_slides.pptx` (para a aula expositiva).

No Encontro 1 construímos a intuição de **pipeline** (Extrair → Transformar → Carregar) em Python puro. No Encontro 2 trouxemos o **pandas** para manipular tabelas com poucas linhas em vez de laços manuais. Hoje damos o passo que faltava: **SQL**, a língua franca dos dados estruturados. SQL não é "mais uma sintaxe" — é uma **forma diferente de pensar**, voltada para conjuntos, que você usará a vida inteira na área, independentemente da linguagem principal do seu projeto.

Para rodar SQL sem instalar um servidor de banco, usamos o **DuckDB**: um banco analítico que roda *dentro do próprio notebook* (in-process), lê DataFrames e arquivos Parquet diretamente, e fala SQL padrão. É o ambiente perfeito para aprender — e, cada vez mais, um ambiente de produção real.

> **Fio condutor do encontro:** o nosso `vendas.csv` dos encontros anteriores vira um pequeno **modelo estrela** sintético — uma tabela-fato `vendas` cercada por duas dimensões, `clientes` e `categorias`. Toda consulta deste encontro acontece sobre esse modelo.

> **Combinado pedagógico:** SQL se aprende escrevendo consultas. A teoria abaixo existe para dar sentido ao código — não para ser decorada.

---

## 1. Pensar em conjuntos: SQL declarativo vs. código procedural

### Definição

**SQL** (*Structured Query Language*) é uma linguagem **declarativa**: você descreve **O QUE** quer obter, e o banco decide **COMO** obtê-lo. Isso é o oposto do código **procedural** que escrevemos em Python no Encontro 1, onde detalhamos o **passo a passo** — abra o arquivo, percorra cada linha, acumule num dicionário, trate o vazio, e assim por diante.

Compare a mesma pergunta — *"qual o faturamento por categoria?"* — nas duas mentalidades:

**Procedural (Python — você descreve o caminho):**

```python
faturamento = {}
for linha in vendas:
    cat = linha["categoria"]
    faturamento[cat] = faturamento.get(cat, 0.0) + linha["valor"]
```

**Declarativo (SQL — você descreve o destino):**

```sql
SELECT categoria, SUM(valor) AS faturamento
FROM vendas
GROUP BY categoria;
```

No SQL não há laço, não há acumulador, não há "primeiro isto, depois aquilo". Você afirma a **forma do resultado** e o banco escolhe o plano de execução — quais índices usar, em que ordem ler, se vale paralelizar. Esse plano é trabalho de um componente chamado **otimizador de consultas**, e ele costuma ser melhor nisso do que nós.

### Por que isso importa para Engenharia de Dados

Pensar em **conjuntos** (set-based), e não em **linhas, uma de cada vez** (row-by-row), é a virada mental central deste encontro. Um engenheiro de dados experiente, diante de "preciso transformar esta tabela", pergunta primeiro: *"isso é uma operação de conjunto que o banco pode fazer por mim?"* — porque empurrar o trabalho para o banco quase sempre é mais rápido, mais legível e mais escalável do que trazer milhões de linhas para a memória e iterar.

> **Pontos de Atenção**
> - SQL declarativo não significa "sem controle": significa que o controle muda de nível — você controla **o resultado**, o otimizador controla **o caminho**.
> - "Pensar linha a linha" é o vício mais comum de quem chega do procedural. Quando notar que está "querendo um `for`" dentro do SQL, provavelmente existe uma operação de conjunto (`GROUP BY`, `JOIN`, window function) que resolve melhor.
> - O mesmo SQL roda em DuckDB, PostgreSQL, BigQuery, Snowflake... A sintaxe padrão é portável; é por isso que SQL atravessa décadas e plataformas.

> 🔗 **Conexão com a Prática:** seção *1 — Pensar em conjuntos* do notebook monta o modelo estrela em DuckDB e responde a mesma pergunta em Python e em SQL, lado a lado.

---

## 2. SELECT essencial: a espinha dorsal de toda consulta

### A anatomia de um SELECT

Quase tudo em SQL começa por `SELECT`. A estrutura mínima e os blocos mais usados são:

```sql
SELECT   coluna1, coluna2, ...      -- O QUE trazer (projeção)
FROM     tabela                     -- DE ONDE
WHERE    condicao                   -- QUAIS linhas (filtro)
ORDER BY coluna [ASC|DESC]          -- EM QUE ordem
LIMIT    n;                         -- QUANTAS linhas
```

Aplicado à nossa tabela-fato:

```sql
SELECT cliente_id, categoria_id, valor, data
FROM vendas
WHERE valor > 100
ORDER BY valor DESC
LIMIT 5;
```

Lê-se naturalmente: *"traga cliente, categoria, valor e data das vendas acima de R$ 100, da maior para a menor, só as cinco primeiras"*. Repare que a **ordem lógica de execução** não é a ordem em que escrevemos: o banco primeiro resolve o `FROM`, depois o `WHERE`, depois projeta o `SELECT`, ordena e por fim aplica o `LIMIT`. Saber disso evita confusões clássicas (por exemplo, por que um apelido criado no `SELECT` às vezes não pode ser usado no `WHERE`).

### Tipos e `CAST`

Todo banco é fortemente tipado: cada coluna tem um tipo (`INTEGER`, `DOUBLE`, `VARCHAR`, `DATE`, `BOOLEAN`...). Quando um dado vem como texto mas precisa ser tratado como número ou data, usamos `CAST`:

```sql
SELECT CAST(valor AS DOUBLE) AS valor_num,
       CAST(data  AS DATE)   AS data_real
FROM vendas;
```

Esse cuidado é o mesmo que tínhamos no Python ao converter `float(linha["valor"])`: **dado bruto chega como texto; análise exige tipos certos.** A diferença é que, em SQL, a conversão é declarada na consulta e o banco a aplica sobre o conjunto inteiro.

### `DISTINCT` — valores únicos

`DISTINCT` resolve em uma palavra o que no Encontro 1 resolvíamos com um `set`:

```sql
SELECT DISTINCT categoria_id FROM vendas;
```

> **Pontos de Atenção**
> - `SELECT *` é cômodo na exploração, mas **evite-o em pipeline**: ele traz colunas que você não controla, quebra contratos quando a fonte muda e desperdiça I/O. Liste as colunas que realmente precisa.
> - `WHERE` filtra **linhas** *antes* da agregação; não confunda com `HAVING` (seção 3), que filtra **grupos** depois.
> - Comparar com `NULL` exige `IS NULL` / `IS NOT NULL` — `= NULL` **nunca** é verdadeiro, porque `NULL` significa "desconhecido", não "vazio".

> 🔗 **Conexão com a Prática:** seção *2 — SELECT essencial* do notebook executa filtros, ordenação, `CAST` e `DISTINCT` sobre a tabela `vendas`.

---

## 3. Agregação: transformar muitas linhas em poucas respostas

### A ideia central

Agregar é **colapsar muitas linhas em um resumo**. É a operação que responde às perguntas de negócio mais comuns: *quanto faturamos, quantos pedidos, qual o ticket médio, qual o maior valor?* O par fundamental é `GROUP BY` + uma **função de agregação**:

| Função | Responde |
|---|---|
| `COUNT(*)` | Quantas linhas? |
| `SUM(col)` | Qual o total? |
| `AVG(col)` | Qual a média? |
| `MIN(col)` / `MAX(col)` | Qual o menor / maior? |

```sql
SELECT categoria_id,
       COUNT(*)      AS qtd_vendas,
       SUM(valor)    AS faturamento,
       AVG(valor)    AS ticket_medio,
       MAX(valor)    AS maior_venda
FROM vendas
GROUP BY categoria_id
ORDER BY faturamento DESC;
```

A regra de ouro: **toda coluna no `SELECT` que não está dentro de uma função de agregação precisa aparecer no `GROUP BY`.** O `GROUP BY` define a granularidade do resultado — uma linha por categoria, no exemplo acima.

### `HAVING` — filtrar grupos

`WHERE` não enxerga agregações, porque roda *antes* delas. Para filtrar pelo resultado de um `SUM` ou `COUNT`, usa-se `HAVING`, que age *depois* do `GROUP BY`:

```sql
SELECT categoria_id, SUM(valor) AS faturamento
FROM vendas
GROUP BY categoria_id
HAVING SUM(valor) > 500;          -- só categorias que faturaram mais de R$ 500
```

### `CASE WHEN` — classificar dentro da consulta

`CASE WHEN` é o `if/elif/else` do SQL. Ele cria categorias derivadas em tempo de consulta — útil para faixas, flags e rótulos:

```sql
SELECT valor,
       CASE
         WHEN valor >= 1000 THEN 'alto valor'
         WHEN valor >= 100  THEN 'valor medio'
         ELSE 'baixo valor'
       END AS faixa
FROM vendas;
```

Combinado com agregação, `CASE WHEN` permite o clássico "soma condicional" (*conditional aggregation*): `SUM(CASE WHEN ... THEN valor ELSE 0 END)`, que pivota dados em uma só passada.

> **Pontos de Atenção**
> - Esquecer uma coluna no `GROUP BY` é o erro nº 1 de iniciantes; o banco recusa a consulta (ou, pior, em alguns dialetos, devolve resultado ambíguo). Em DuckDB, ele recusa — e isso é um favor.
> - `COUNT(*)` conta linhas; `COUNT(coluna)` **ignora `NULL`** naquela coluna. A diferença muda o número e já causou muito relatório errado.
> - `AVG` também ignora `NULL` — uma média pode "esconder" buracos no dado. Saiba quantos `NULL` existem antes de confiar na média.

> 🔗 **Conexão com a Prática:** seção *3 — Agregação* do notebook calcula faturamento, ticket médio e contagem por categoria, filtra com `HAVING` e classifica vendas com `CASE WHEN`.

---

## 4. JOINs: juntar a fato com as dimensões

### Por que separar em fato e dimensões

No nosso modelo estrela, a tabela `vendas` (a **fato**) guarda apenas *o que aconteceu* e referências numéricas: `cliente_id`, `categoria_id`, `valor`, `data`. Os detalhes descritivos vivem nas **dimensões**: `clientes` (nome, cidade, segmento) e `categorias` (nome, departamento). Essa separação evita repetir o nome do cliente em cada linha de venda e é a base da modelagem dimensional. Para responder *"quanto cada cliente comprou, por nome"*, precisamos **costurar** fato e dimensão — isso é um `JOIN`.

### O `JOIN` e suas variações

Um `JOIN` combina linhas de duas tabelas com base em uma condição (geralmente igualdade de chaves). O que muda entre os tipos é **o que acontece com as linhas que não encontram par**:

```sql
SELECT v.id, c.nome AS cliente, v.valor
FROM vendas v
INNER JOIN clientes c ON v.cliente_id = c.id;
```

| Tipo | O que preserva |
|---|---|
| `INNER JOIN` | Só as linhas com correspondência **nas duas** tabelas. |
| `LEFT JOIN` | **Todas** as da esquerda; preenche com `NULL` quando não há par à direita. |
| `RIGHT JOIN` | **Todas** as da direita; espelho do `LEFT`. |
| `FULL [OUTER] JOIN` | **Todas** de ambos os lados; `NULL` onde faltar par em qualquer um. |

A escolha do tipo é uma **decisão de engenharia**, não de estilo. *"Quero faturamento por cliente, mas preciso ver clientes que não compraram nada"* → `LEFT JOIN` da dimensão `clientes` com a fato `vendas` (os sem compra aparecem com faturamento `NULL`). *"Quero detectar vendas órfãs apontando para um cliente que não existe mais"* → um `LEFT JOIN` cujo lado direito vem `NULL` denuncia a inconsistência. O `INNER JOIN` é o padrão quando você só quer o que casa dos dois lados.

> **Pontos de Atenção**
> - Um `JOIN` com condição errada (ou ausente) gera um **produto cartesiano**: cada linha de uma tabela cruza com *todas* da outra. Em tabelas grandes, isso explode e é uma fonte clássica de "consulta que travou o banco".
> - `LEFT JOIN` seguido de `WHERE coluna_direita = algo` **silenciosamente vira um `INNER JOIN`**, porque o filtro elimina os `NULL` que o `LEFT` tinha preservado. Para manter o comportamento de `LEFT`, filtre dentro do `ON` ou teste `IS NULL`.
> - Use **apelidos** de tabela (`vendas v`, `clientes c`) e sempre **qualifique** as colunas (`v.valor`) — em consultas com vários `JOIN`, isso evita ambiguidade e torna o SQL legível.

> 🔗 **Conexão com a Prática:** seção *4 — JOINs* do notebook demonstra os quatro tipos juntando a fato `vendas` às dimensões `clientes` e `categorias`, e mostra o que cada um preserva.

---

## 5. Window functions e CTEs: ranking e legibilidade

### O problema que as window functions resolvem

Agregação (`GROUP BY`) **colapsa** linhas: você perde o detalhe e fica só com o resumo. Mas muitas perguntas exigem o resumo **junto** do detalhe: *"qual a posição deste cliente no ranking de faturamento?"*, *"quanto esta venda representa do total da sua categoria?"*, *"qual o acumulado até esta data?"*. As **window functions** (funções de janela) calculam um valor agregado **sem colapsar as linhas** — cada linha continua existindo, agora acompanhada do seu resultado de janela.

A estrutura é uma função seguida da cláusula `OVER`:

```sql
funcao() OVER (PARTITION BY coluna ORDER BY coluna)
```

- `PARTITION BY` define os **grupos** (a "janela") — análogo ao `GROUP BY`, mas sem fundir as linhas.
- `ORDER BY` (dentro do `OVER`) define a **ordem** dentro de cada janela, necessária para rankings e acumulados.

Funções comuns:

```sql
ROW_NUMBER() OVER (ORDER BY faturamento DESC)   -- posição única: 1,2,3,4...
RANK()       OVER (ORDER BY faturamento DESC)   -- empates dividem a posição: 1,2,2,4
SUM(valor)   OVER (PARTITION BY categoria_id)   -- total da categoria em cada linha
```

### CTEs: dar nome às etapas com `WITH`

Uma **CTE** (*Common Table Expression*), introduzida por `WITH`, é uma consulta nomeada que serve de "tabela temporária" dentro da consulta principal. Ela não muda o resultado — muda a **legibilidade**: em vez de subconsultas aninhadas e ilegíveis, você nomeia cada etapa do raciocínio, de cima para baixo, como funções pequenas.

O exemplo canônico do encontro — **ranking de clientes por faturamento** — junta as duas ideias:

```sql
WITH faturamento_cliente AS (
    SELECT c.nome,
           SUM(v.valor) AS faturamento
    FROM vendas v
    JOIN clientes c ON v.cliente_id = c.id
    GROUP BY c.nome
)
SELECT nome,
       faturamento,
       RANK() OVER (ORDER BY faturamento DESC) AS posicao
FROM faturamento_cliente
ORDER BY posicao;
```

A CTE `faturamento_cliente` resolve o "quanto cada cliente comprou"; a consulta final apenas **ranqueia** o resultado. Cada etapa faz uma coisa, tem nome e pode ser lida isoladamente. É exatamente o mesmo princípio de decompor um pipeline em funções pequenas — agora em SQL.

> **Pontos de Atenção**
> - Não confunda `WHERE` com janela: você **não pode** filtrar diretamente por uma window function no `WHERE` (ela é calculada depois). Para "pegar o top-N por grupo", calcule a janela em uma CTE e filtre na consulta seguinte (`WHERE posicao <= 3`).
> - `ROW_NUMBER` sempre dá posições únicas (desempata arbitrariamente); `RANK` repete a posição em empates e "pula" números; `DENSE_RANK` repete sem pular. Escolha conforme a regra de negócio do ranking.
> - CTEs custam (quase) nada em legibilidade e são o melhor antídoto contra "consultas-monstro". Quando uma subconsulta começar a aninhar, promova-a a CTE.

> 🔗 **Conexão com a Prática:** seção *5 — Window functions e CTEs* do notebook constrói o ranking de clientes por faturamento usando `WITH` + `RANK()`.

---

## 6. SQL + Python na prática: DuckDB sobre DataFrames e Parquet

### O melhor dos dois mundos

Na vida real, você raramente escolhe "ou Python ou SQL". Você os **combina**: usa Python para orquestrar, integrar e tratar o que SQL não faz bem, e usa SQL para o trabalho pesado de filtrar, agregar e juntar conjuntos. O DuckDB torna essa combinação trivial, porque consulta **DataFrames pandas e arquivos diretamente**, sem precisar carregar tudo numa tabela antes.

**Consultar um DataFrame pandas que está na memória** — o DuckDB enxerga a variável Python pelo nome:

```python
import duckdb, pandas as pd
df = pd.read_csv("vendas.csv")           # DataFrame na memória
duckdb.sql("SELECT categoria_id, SUM(valor) FROM df GROUP BY categoria_id").df()
```

**Registrar explicitamente** um DataFrame (útil quando o nome da tabela difere da variável, ou em conexões persistentes):

```python
con = duckdb.connect()
con.register("vendas", df)
con.sql("SELECT * FROM vendas WHERE valor > 100").df()
```

**Ler Parquet direto do disco**, sem ETL prévio — o DuckDB lê só as colunas e linhas que a consulta exige:

```python
duckdb.sql("SELECT categoria_id, SUM(valor) FROM read_parquet('vendas.parquet') GROUP BY categoria_id").df()
```

O **Parquet** é o formato colunar padrão do mundo analítico: comprime bem, guarda os tipos e é lido de forma eficiente por colunas. A dupla **DuckDB + Parquet** permite analisar arquivos grandes com SQL sem subir banco nenhum.

### Quando empurrar o trabalho para o banco

Uma heurística prática para decidir onde fazer cada coisa:

- **Empurre para o SQL/banco:** filtros, agregações, junções, ordenações sobre volumes grandes. O banco foi construído para isso, processa em formato colunar e move menos dados.
- **Mantenha em Python:** orquestração, lógica condicional complexa, chamadas a APIs, integração com bibliotecas (ML, requests), e o "encanamento" entre etapas.

A regra que resume o encontro: **filtre e agregue o mais cedo possível, o mais perto da fonte possível.** Trazer um milhão de linhas para a memória só para somá-las é desperdício; peça a soma ao banco e traga uma linha.

> **Caixa de tendência de mercado — DuckDB e dbt**
>
> **DuckDB** vem sendo chamado de *"o SQLite do OLAP"*: um banco analítico in-process, sem servidor, que roda dentro do seu script e consulta Parquet/CSV/DataFrames diretamente. Ele está redefinindo o que cabe numa "máquina só" — análises que antes exigiam um cluster hoje rodam num notebook. É a ferramenta que usamos o encontro inteiro.
>
> **dbt** (*data build tool*) leva o SQL para o mundo da engenharia de software: cada transformação vira um **model** — um arquivo `.sql` versionado no Git, modular, testável e documentado. Em vez de scripts soltos, você tem um grafo de dependências de transformações, com testes automáticos de qualidade e documentação gerada. Um model dbt ilustrativo (`faturamento_por_cliente.sql`):
>
> ```sql
> -- models/marts/faturamento_por_cliente.sql
> with vendas as (
>     select * from {{ ref('stg_vendas') }}
> ),
> clientes as (
>     select * from {{ ref('stg_clientes') }}
> )
> select
>     c.nome as cliente,
>     count(*)        as qtd_vendas,
>     sum(v.valor)    as faturamento
> from vendas v
> join clientes c on v.cliente_id = c.id
> group by c.nome
> ```
>
> O `{{ ref(...) }}` referencia outro model e deixa o dbt montar a ordem de execução sozinho. A tendência é clara: **transformação de dados é SQL versionado, modular e testado** — não consulta avulsa colada num agendador.

> 🔗 **Conexão com a Prática:** seção *6 — SQL + Python* do notebook consulta um DataFrame pandas e um arquivo Parquet com DuckDB, e discute o que vale empurrar para o banco.

---

## Síntese do encontro

Hoje trocamos a lente: do "passo a passo" procedural para o "o que eu quero" declarativo.

1. **Pensar em conjuntos** é a virada mental do SQL: você descreve o resultado; o otimizador escolhe o caminho. Empurrar trabalho para o banco vence iterar na memória.
2. **`SELECT` essencial** (`WHERE`, `ORDER BY`, `LIMIT`, `CAST`, `DISTINCT`) é a espinha dorsal de toda consulta — e `NULL` exige cuidado especial.
3. **Agregação** (`GROUP BY`, `HAVING`, `SUM/AVG/COUNT/MIN/MAX`, `CASE WHEN`) colapsa muitas linhas em poucas respostas de negócio.
4. **JOINs** costuram a fato às dimensões; o tipo (`INNER/LEFT/RIGHT/FULL`) é uma decisão de engenharia sobre **o que preservar** quando não há par.
5. **Window functions e CTEs** trazem resumo *sem* perder o detalhe (ranking, acumulado) e tornam consultas complexas legíveis com `WITH`.
6. **DuckDB + Python** combinam as duas linguagens: SQL para o trabalho pesado sobre DataFrames e Parquet; Python para a orquestração. A tendência (**DuckDB**, **dbt**) é tratar transformação como SQL versionado e testado.

### Para o próximo encontro

- Rode o notebook inteiro; reescreva pelo menos uma consulta procedural do Encontro 1 como SQL declarativo.
- Pense num relatório do seu trabalho que hoje você monta "puxando tudo para uma planilha e cruzando à mão". Identifique os **JOINs** e as **agregações** escondidas nele — vamos discutir.
- No **Encontro 4** saímos do "uma máquina só": entramos em **processamento em escala** (o que muda quando o dado não cabe na memória, e como motores distribuídos respondem a isso).

---

## Bibliografia

- MCKINNEY, Wes. *Python para Análise de Dados*. 3ª ed. O'Reilly, 2022.
- REIS, Joe; HOUSLEY, Matt. *Fundamentos de Engenharia de Dados*. O'Reilly / Alta Books, 2023.
- KLEPPMANN, Martin. *Designing Data-Intensive Applications*. O'Reilly, 2017.

---

*Material de apoio — UNIFOR Pós-Graduação · Prof. Cassio Pinheiro. Acompanha o notebook prático e os slides da disciplina.*
