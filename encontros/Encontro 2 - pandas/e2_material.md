# Linguagens de Programação para Engenharia de Dados
## Encontro 2 — Manipulação de dados com pandas

**Instituição:** Universidade de Fortaleza (UNIFOR)
**Curso:** Pós-Graduação Lato Sensu — Especialização em Engenharia de Dados
**Professor:** Cassio Pinheiro · [LinkedIn](https://www.linkedin.com/in/cassio-pinheiro-9baa7939/)
**Data:** 26/06/2026 · 19:00 às 22:30
**Carga horária da disciplina:** 24h

---

## Como ler este documento

Este é o material de fundamentação do **segundo encontro**. Ele acompanha:

- o **notebook** `e2_pratica.ipynb` (onde rodamos código — cada seção do notebook é referenciada aqui com a marca 🔗);
- os **slides** `e2_slides.pptx` (para a aula expositiva).

No Encontro 1 construímos, com **Python puro**, um mini-pipeline ETL sobre um `vendas.csv` deliberadamente sujo: laços, condicionais, dicionários acumuladores. Funcionou — e foi pedagógico justamente por mostrar, à mão, cada decisão de limpeza. Mas reescrever aquele código todo dia, sobre milhões de linhas, é inviável. Hoje damos o salto: o **mesmo problema, resolvido com pandas**, a biblioteca que é o padrão de fato para manipulação de dados tabulares em Python. Vamos refazer o pipeline do E1 em poucas linhas, gerar um arquivo **Parquet** confiável, e — importante — chegar **exatamente ao mesmo resultado de faturamento** do encontro anterior. Mesma água, encanamento melhor.

> **Combinado pedagógico:** não trocamos Python puro por pandas porque pandas é "mais moderno". Trocamos porque ele *expressa a intenção* (limpar, agrupar, juntar) no nível de tabela, e executa por baixo em código compilado. Entender o porquê é tão importante quanto a sintaxe.

---

## 1. Por que pandas: do Python puro ao DataFrame

### Definição

**pandas** é uma biblioteca de código aberto para manipulação e análise de dados estruturados em Python, criada por Wes McKinney. Sua estrutura central é o **DataFrame**: uma tabela em memória, com **colunas nomeadas e tipadas** e um **índice** que identifica as linhas. Se no E1 representamos uma tabela como *lista de dicionários*, o DataFrame é a versão industrial dessa ideia — só que com operações de conjunto, tipos explícitos e desempenho de outra ordem de grandeza.

### Vetorização: por que é rápido

A diferença essencial entre o E1 e o pandas não é estética, é arquitetural. No Python puro, somar uma coluna significa **iterar linha a linha** num laço interpretado:

```python
soma = 0.0
for v in valores:        # laço Python, interpretado, lento
    soma += v
```

No pandas, a mesma soma é uma **operação vetorizada**: você descreve a operação sobre a coluna inteira de uma vez (`df["valor"].sum()`), e por baixo o pandas a executa em **C otimizado sobre arrays NumPy contíguos na memória**, sem voltar ao interpretador Python a cada elemento. Isso é *vetorização*. Na prática, uma agregação vetorizada sobre milhões de linhas costuma ser **dezenas a centenas de vezes mais rápida** que o laço equivalente — não porque "C é rápido" no abstrato, mas porque elimina o overhead de interpretar bytecode a cada iteração e aproveita melhor o cache da CPU.

A regra mental para o curso inteiro: **em pandas, evite o `for` sobre linhas.** Quase sempre existe uma operação de coluna que faz o mesmo trabalho mais rápido e mais legível. O laço manual do E1 foi um andaime didático; a partir daqui, pensamos em **colunas inteiras**.

> **Pontos de Atenção**
> - pandas brilha no que cabe na **memória de uma máquina**. Para volumes que estouram a RAM, entram motores como Spark, Polars (modo lazy) ou processamento em lote — assunto de encontros futuros.
> - "Vetorizar" não é só evitar o `for`: é usar as operações nativas de coluna (`.sum()`, máscaras, `groupby`) em vez de `.apply(lambda ...)` linha a linha, que muitas vezes recai no laço lento por baixo.
> - O ganho de produtividade é tão grande quanto o de desempenho: o que eram 20 linhas de laço no E1 vira 3 ou 4 linhas declarativas.

> 🔗 **Conexão com a Prática:** seção *1 — Por que pandas* do notebook mede, no mesmo dado, o laço Python puro contra a soma vetorizada do pandas.

---

## 2. DataFrame e Series: a tabela e a coluna

### Definição

Há duas estruturas que sustentam tudo em pandas:

- **Series** — uma **coluna**: uma sequência unidimensional de valores, todos do mesmo tipo lógico, com um índice. Uma coluna de um DataFrame é uma Series.
- **DataFrame** — uma **tabela**: um conjunto de Series alinhadas pelo mesmo índice. É o objeto com que você passa 90% do tempo.

### Leitura: CSV, JSON e Parquet

pandas lê de praticamente qualquer fonte com uma função `read_*`. As três que mais aparecem em Engenharia de Dados:

```python
df = pd.read_csv("vendas.csv")          # texto, universal, mas sem tipos
df = pd.read_json("eventos.json")       # comum em respostas de API
df = pd.read_parquet("vendas.parquet")  # colunar, comprimido, COM tipos
```

A diferença entre eles não é detalhe: **CSV é texto puro e não carrega tipos** — toda coluna chega como string e você precisa converter. **Parquet é um formato colunar binário** que guarda o schema (nomes e tipos das colunas) junto com os dados, é comprimido e é lido muito mais rápido. Por isso, no mundo real, o CSV é o que *chega* da fonte e o Parquet é o que você *entrega* ao consumidor — exatamente o que faremos na seção 6.

### `dtypes` e por que importam

Cada coluna de um DataFrame tem um **dtype** (tipo de dado): `int64`, `float64`, `object` (geralmente texto), `datetime64[ns]`, `bool`, `category`. O dtype não é burocracia: ele determina **quais operações fazem sentido e como o dado ocupa memória**. Somar uma coluna que ficou como `object` (texto) ou não funciona, ou concatena strings em vez de somar números. Metade dos bugs de limpeza nasce de um dtype errado lido de um CSV.

### Inspeção: `head`, `info`, `describe`

Três métodos que você roda assim que carrega qualquer dado, antes de qualquer transformação:

- `df.head()` — espia as primeiras linhas. "Com o que estou lidando?"
- `df.info()` — colunas, dtypes, contagem de não-nulos, uso de memória. "Os tipos estão certos? Onde há buracos?"
- `df.describe()` — estatísticas (média, min, max, quartis) das colunas numéricas. "Há um valor negativo ou absurdo escondido aqui?"

> **Pontos de Atenção**
> - Ao ler um CSV, **sempre confira `df.info()`** logo em seguida: é onde você descobre que `valor` virou `object` por causa de um campo vazio, ou que a data continua sendo texto.
> - `object` quase nunca é o tipo que você quer para números ou datas — é o sinal de "ainda precisa converter".
> - `describe()` é um detector barato de problema: um `min` negativo numa coluna de valores, ou um `max` absurdo, aparecem aqui antes de contaminarem um relatório.

> 🔗 **Conexão com a Prática:** seção *2 — DataFrame e Series* do notebook lê o `vendas.csv` sujo, mostra `head/info/describe` e expõe por que os dtypes ainda estão "errados" no dado bruto.

---

## 3. Seleção e filtragem: pegar exatamente o que interessa

### `loc` e `iloc`

Selecionar subconjuntos é a operação mais frequente em qualquer análise. pandas oferece dois acessadores complementares:

- **`.loc[linhas, colunas]`** — seleção por **rótulo** (nome do índice e nome da coluna). `df.loc[df["valor"] > 100, ["cliente", "valor"]]`.
- **`.iloc[linhas, colunas]`** — seleção por **posição inteira** (como índices de lista, começando em 0). `df.iloc[0:3, :]` pega as três primeiras linhas.

A regra: use `.loc` quando pensar em "qual coluna/condição" (o caso comum em dados), e `.iloc` quando pensar em "qual posição".

### Seleção de colunas

```python
df["valor"]              # uma coluna -> Series
df[["cliente", "valor"]] # várias colunas -> DataFrame
```

### Máscaras booleanas: o coração da filtragem

Aqui está a ideia mais poderosa — e mais vetorizada — do pandas. Uma comparação sobre uma coluna **não devolve um valor, devolve uma Series de `True`/`False`** (uma *máscara*), com um booleano por linha:

```python
mascara = df["valor"] > 100      # Series de True/False, uma por linha
df[mascara]                       # devolve só as linhas onde a máscara é True
```

Você combina condições com `&` (e), `|` (ou) e `~` (não), **sempre entre parênteses**:

```python
df[(df["valor"] > 0) & (df["categoria"] == "eletronicos")]
```

Compare isto com o E1: lá, filtrar "valor > 0 e categoria X" exigia um `if` dentro de um `for`. Aqui é uma expressão declarativa que diz *o que* queremos, e o pandas decide *como* percorrer — vetorizado, em C.

> **Pontos de Atenção**
> - Em máscaras, use `&`, `|`, `~` — **não** os operadores `and`, `or`, `not` do Python (eles operam sobre um booleano único, não sobre a Series inteira) — e **envolva cada condição em parênteses**, por causa da precedência.
> - `df[df["x"] > 0]` cria uma *visão/cópia*; para alterar o original com segurança, prefira `df.loc[mascara, "coluna"] = valor`, evitando o aviso `SettingWithCopyWarning`.
> - Filtrar cedo (reduzir linhas antes de transformar) é também uma decisão de desempenho: menos dados, menos trabalho rio abaixo.

> 🔗 **Conexão com a Prática:** seção *3 — Seleção e filtragem* do notebook isola, com máscaras, exatamente as linhas problemáticas (valor negativo, ausente) que vamos tratar na seção seguinte.

---

## 4. Limpeza com pandas: o pipeline do E1 em operações de tabela

### Definição

Limpar é converter dado bruto em dado confiável (os critérios de "confiável" do E1 continuam valendo: completo, consistente, pontual, rastreável). O pandas dá um verbo de tabela para cada problema que tratamos à mão no E1.

### Valores ausentes: `isna`, `fillna`, `dropna`

pandas representa ausência com `NaN` (*Not a Number*) ou `NaT` (datas). Você os detecta e trata com:

```python
df["valor"].isna()             # máscara: onde está ausente?
df["valor"].fillna(0)          # preenche ausentes com um valor
df.dropna(subset=["valor"])    # remove linhas ausentes em 'valor'
```

A escolha entre **preencher** (`fillna`) e **remover** (`dropna`) é uma decisão de negócio, não técnica: um valor de venda ausente provavelmente deve sair (não inventamos receita); um campo de comentário ausente pode virar string vazia. Decida e **documente**.

### Conversão de tipos: `astype`, `to_datetime`, `to_numeric` com `errors`

O calcanhar de Aquiles do CSV. Três ferramentas:

```python
df["valor"] = pd.to_numeric(df["valor"], errors="coerce")    # texto -> número; inválido vira NaN
df["data"]  = pd.to_datetime(df["data"], errors="coerce",     # texto -> data
                             format="mixed", dayfirst=True)    # lida com formatos mistos
df["id"]    = df["id"].astype("int64")                         # conversão direta
```

O parâmetro **`errors="coerce"`** é a peça-chave de robustez: em vez de **quebrar o pipeline** ao encontrar um valor que não converte, ele transforma o inválido em `NaN`/`NaT`. Isso converte "erro fatal" em "ausência tratável" — exatamente a filosofia de tratamento de erro do E1, agora declarativa. As datas do nosso CSV vêm em três formatos (`AAAA-MM-DD`, `DD/MM/AAAA`, `AAAA/MM/DD`); `to_datetime` com `format="mixed"` e `dayfirst=True` resolve os três e marca a data vazia como `NaT`.

### Duplicatas e espaços: `drop_duplicates`, `str.strip`

```python
df = df.drop_duplicates()                     # remove linhas idênticas (id 5)
df["cliente"] = df["cliente"].str.strip()     # remove espaços das pontas (id 2)
```

O acessador **`.str`** aplica métodos de string a uma coluna inteira de forma vetorizada — `str.strip()`, `str.lower()`, `str.replace()` — sem laço.

> **Pontos de Atenção**
> - `errors="coerce"` é poderoso mas **silencioso**: depois de converter, **conte os `NaN` gerados** (`df["valor"].isna().sum()`). Um silêncio não auditado é a origem de erros que aparecem semanas depois — a mesma lição de rastreabilidade do E1.
> - `drop_duplicates()` por padrão considera **todas** as colunas; use `subset=[...]` para definir a chave de duplicação que faz sentido no negócio.
> - A maioria dos métodos de limpeza **devolve um novo DataFrame** e não altera o original (não é *in place*). Atribua o resultado de volta (`df = df.dropna(...)`) ou encadeie.

> 🔗 **Conexão com a Prática:** seção *4 — Limpeza com pandas* do notebook trata, uma a uma, todas as sujeiras do `vendas.csv` (ausente, negativo, duplicata, data em 3 formatos, espaços), contando cada descarte.

---

## 5. Transformação e agregação: derivar, agrupar e juntar

### Colunas derivadas: `assign`

Enriquecer é criar colunas novas a partir das existentes — vetorizado:

```python
df = df.assign(total_com_imposto = df["valor"] * 1.18)
```

`assign` devolve um novo DataFrame com a coluna adicionada e encadeia bem; o equivalente direto `df["total_com_imposto"] = df["valor"] * 1.18` também funciona. Repare: nenhuma iteração — a multiplicação acontece sobre a coluna inteira de uma vez.

### Agregação: `groupby().agg()`

O `groupby` é o `GROUP BY` do SQL trazido para Python, e responde a pergunta central do nosso fio condutor — *"qual o faturamento por categoria?"* — em uma linha:

```python
df.groupby("categoria")["valor"].sum()
df.groupby("categoria").agg(
    faturamento=("valor", "sum"),
    ticket_medio=("valor", "mean"),
    qtd=("valor", "count"),
)
```

O modelo mental é **split → apply → combine**: o pandas *divide* o DataFrame por categoria, *aplica* a agregação a cada grupo e *combina* num resultado. É a versão vetorizada e legível do dicionário acumulador que escrevemos à mão no E1.

### Junção: `merge` / join

Dados reais vivem espalhados em várias tabelas. O **`merge`** junta DataFrames por uma chave comum — é o `JOIN` do SQL:

```python
df_enriquecido = df.merge(dim_categoria, on="categoria", how="left")
```

O parâmetro `how` define o tipo de junção (`left`, `inner`, `outer`, `right`), a mesma semântica do SQL que veremos no E3. Usaremos um `left` join com uma pequena **tabela de dimensão** que classifica cada categoria como "essencial" ou "supérfluo" — o padrão fato × dimensão, base de toda modelagem analítica.

### Tabela dinâmica: `pivot_table`

`pivot_table` reorganiza dados longos em uma matriz (linhas × colunas × valor agregado), como a tabela dinâmica do Excel — útil para visões de negócio (ex.: faturamento por categoria × dia).

> **Pontos de Atenção**
> - Em `merge`, confira o **número de linhas antes e depois**: um join mal-feito (chave duplicada do lado errado) **multiplica linhas** silenciosamente e infla qualquer soma posterior.
> - `groupby` por padrão **ignora `NaN`** na chave de agrupamento — o que costuma ser desejável, mas precisa ser consciente.
> - Prefira a sintaxe nomeada do `agg` (`faturamento=("valor", "sum")`): produz colunas com nomes claros, e pipeline legível é pipeline manutenível.

> 🔗 **Conexão com a Prática:** seção *5 — Transformação e agregação* do notebook cria colunas derivadas, agrupa o faturamento por categoria, faz um `merge` com uma dimensão e monta uma `pivot_table`.

---

## 6. O pipeline do E1 refeito em pandas

### O objetivo do encontro converge aqui

Vamos pegar o **mesmo `vendas.csv` sujo** do Encontro 1 e levá-lo de bruto a confiável usando tudo que vimos — agora em poucas linhas declarativas, no padrão **Extrair → Transformar → Carregar**. A prova de que fizemos certo é simples: o **faturamento por categoria do dado tratado tem que bater exatamente com o do E1** (4 linhas confiáveis: eletrônicos R$ 1200,00; vestuário R$ 260,00; alimentação R$ 128,40).

### O pipeline, em pandas

```python
df = (
    pd.read_csv(io.StringIO(csv_bruto), dtype=str)            # EXTRAIR (tudo como texto)
      .assign(
          valor=lambda d: pd.to_numeric(d["valor"], errors="coerce"),
          data=lambda d: pd.to_datetime(d["data"], errors="coerce",
                                        format="mixed", dayfirst=True),
          cliente=lambda d: d["cliente"].str.strip(),
      )
      .drop_duplicates()                                      # remove a duplicata (id 5)
      .dropna(subset=["valor", "data"])                       # remove ausente e data vazia
      .query("valor >= 0")                                    # remove o negativo (id 4)
      .assign(total_com_imposto=lambda d: (d["valor"] * 1.18).round(2))  # enriquece
)
df.to_parquet("vendas_tratadas.parquet")                      # CARREGAR
```

Repare na densidade: o que no E1 foram dezenas de linhas de laços, `if`s e dicionários virou **um encadeamento (*method chaining*) declarativo**. Cada `.metodo()` é um passo do pipeline, lido de cima para baixo como uma receita: ler → converter tipos → desduplicar → remover ausências → filtrar negativos → enriquecer → salvar.

### Por que salvar em Parquet (e não CSV)

No E1 entregamos um CSV. Aqui entregamos **Parquet** porque ele preserva os dtypes que tanto trabalho deu para acertar (data continua data, valor continua número — o próximo consumidor não precisa re-converter), é **comprimido** (ocupa menos espaço) e **colunar** (ler só a coluna `valor` não exige varrer o arquivo inteiro). É o formato padrão de entrega em data lakes modernos.

> **Caixa de tendência de mercado — Arrow e Polars**
>
> Duas tendências estão redesenhando esse cenário, e vale conhecê-las desde já:
>
> - **Apache Arrow como backend do pandas (2.x).** Historicamente o pandas guardava dados em arrays NumPy, com tratamento de ausência e de texto pouco eficiente. A partir do pandas 2.0 é possível usar o **Arrow** como backend de memória — `pd.read_csv(..., dtype_backend="pyarrow")` — um formato **colunar** que representa texto e nulos de forma mais eficiente, acelera a leitura/escrita de Parquet (que já é Arrow por baixo) e melhora a interoperabilidade com outras ferramentas do ecossistema (Spark, DuckDB, Polars). É o pandas que você já conhece, com fundação de memória mais moderna.
>
> - **Polars como alternativa.** O **Polars** é um DataFrame escrito em Rust, **multicore por padrão** e com um modo **lazy**: em vez de executar cada operação na hora, ele monta um *plano de consulta*, **otimiza** o plano inteiro (empurra filtros para perto da leitura, descarta colunas não usadas) e só então executa — o que rende ganhos expressivos em volumes grandes. Sua API é parecida o suficiente com a do pandas para a transição ser suave. No notebook fazemos a **mesma agregação** em pandas e em Polars, lado a lado, sobre o nosso `vendas.csv` — para você sentir a semelhança conceitual e a diferença de filosofia (eager vs. lazy).
>
> A mensagem não é "abandone o pandas". É: o **modelo mental de DataFrame** que você domina aqui é transferível, e o ecossistema está convergindo para memória colunar (Arrow) e execução otimizada (lazy). Quem entende o conceito troca de ferramenta sem trauma.

> **Pontos de Atenção**
> - *Method chaining* é elegante, mas **depure por partes**: rode o encadeamento até cada `.metodo()` e confira o `shape`/`info` antes de adicionar o próximo passo.
> - Para escrever Parquet, o pandas precisa de um motor instalado (`pyarrow`); por isso ele entra nas dependências do notebook.
> - O teste final do pipeline **não é "rodou sem erro"**, é "o número bate com o esperado". Sempre tenha um valor de conferência (aqui, o faturamento do E1) para validar o resultado.

> 🔗 **Conexão com a Prática:** seção *6 — Pipeline do E1 refeito em pandas* do notebook executa o encadeamento completo, salva `vendas_tratadas.parquet`, relê o Parquet conferindo os dtypes e prova que o faturamento bate com o do E1.

---

## Síntese do encontro

Hoje trocamos o motor sem trocar o destino:

1. **pandas e o DataFrame** levam a "lista de dicionários" do E1 ao nível industrial: colunas nomeadas, tipadas, com **vetorização** (operações em C sobre arrays) no lugar de laços interpretados.
2. **DataFrame e Series, `dtypes`, `head/info/describe`** são o ritual de toda chegada de dado — e o lugar onde os problemas aparecem primeiro.
3. **Seleção e filtragem** com `loc`/`iloc` e **máscaras booleanas** substituem os `if` dentro de `for` por expressões declarativas.
4. **Limpeza** (`isna/fillna/dropna`, `to_numeric`/`to_datetime` com `errors="coerce"`, `drop_duplicates`, `str.strip`) trata cada sujeira do CSV como uma operação de tabela.
5. **Transformação e agregação** (`assign`, `groupby().agg()`, `merge`, `pivot_table`) derivam, agrupam e juntam — o `GROUP BY` e o `JOIN` do SQL, em Python.
6. O **pipeline do E1 refeito** vira um encadeamento de poucas linhas, entrega **Parquet** e chega ao **mesmo resultado** — com Arrow e Polars apontando para onde o ecossistema caminha.

### Para o próximo encontro

- Rode o notebook inteiro; quebre uma etapa do encadeamento de propósito e veja o erro mudar de lugar.
- No **Encontro 3 vamos a SQL**: a mesma pergunta de faturamento, agora deixando o banco fazer o trabalho pesado. Repare, ao longo deste notebook, em como `groupby`, `merge` e máscaras já têm um par exato em SQL (`GROUP BY`, `JOIN`, `WHERE`) — essa ponte é o objetivo. Traga uma dúvida concreta de algum dado tabular do seu trabalho.

---

## Bibliografia

- McKINNEY, Wes. *Python para Análise de Dados*. 3ª ed. O'Reilly, 2022.
- REIS, Joe; HOUSLEY, Matt. *Fundamentos de Engenharia de Dados*. O'Reilly / Alta Books, 2023.
- KLEPPMANN, Martin. *Designing Data-Intensive Applications*. O'Reilly, 2017.

---

*Material de apoio — UNIFOR Pós-Graduação · Prof. Cassio Pinheiro. Acompanha o notebook prático e os slides da disciplina.*
