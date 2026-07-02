# Linguagens de Programação para Engenharia de Dados
## Encontro 1 — Fundamentos: o que é Engenharia de Dados e por que programar

**Instituição:** Universidade de Fortaleza (UNIFOR)
**Curso:** Pós-Graduação Lato Sensu — Especialização em Engenharia de Dados
**Professor:** Cassio Pinheiro · [LinkedIn](https://www.linkedin.com/in/cassio-pinheiro-9baa7939/)
**Data:** 25/06/2026 · 19:00 às 22:30
**Carga horária da disciplina:** 24h

---

## Como ler este documento

Este é o material de fundamentação do **primeiro encontro**. Ele acompanha:

- o **notebook** `linguagens_prog_eng_dados_pratica.ipynb` (onde rodamos código — cada seção do notebook é referenciada aqui com a marca 🔗);
- os **slides** `linguagens_prog_eng_dados_slides.pptx` (para a aula expositiva).

A disciplina é uma das **primeiras do curso** e parte do princípio de que a turma é **heterogênea**: há quem programe há anos e há quem venha de contabilidade, administração, engenharia, saúde ou direito. Por isso este encontro é deliberadamente **conceitual e nivelador**. Não vamos otimizar Spark hoje; vamos construir o vocabulário e a intuição que sustentam todo o resto do curso. Quem já programa vai reforçar fundamentos e ver o mesmo conteúdo pela lente de *dados em escala*; quem não programa vai sair daqui com o primeiro pipeline rodando.

> **Combinado pedagógico:** programação se aprende programando. A teoria abaixo existe para dar sentido ao código — não para ser decorada.

---

## 1. O que é Engenharia de Dados (e como difere de Ciência de Dados)

### Definição

**Engenharia de Dados** é a disciplina que projeta, constrói e mantém os **sistemas que movem e preparam dados** para que outras pessoas — cientistas de dados, analistas, áreas de negócio, modelos de IA — possam usá-los com confiança. Reis e Housley resumem o ciclo em cinco estágios: **geração → ingestão → armazenamento → transformação → entrega (serving)**. O engenheiro de dados é quem mantém esse fluxo funcionando de forma confiável, escalável e rastreável.

A confusão com Ciência de Dados é comum, então vale fixar a fronteira com uma analogia:

> Se os dados fossem água, o **engenheiro de dados** constrói o encanamento, as estações de tratamento e as caixas-d'água — garante que a água chegue limpa, na pressão certa, na hora certa. O **cientista de dados** é quem analisa a água para descobrir o que ela revela. Sem encanamento, não há análise.

### A famosa "Hierarquia de Necessidades de IA"

Existe uma pirâmide muito citada na área (originalmente proposta por Monica Rogati): na base estão **coleta e infraestrutura de dados**; só mais acima vêm limpeza, transformação, métricas, e — bem no topo — Machine Learning e IA. A mensagem é direta: **a maior parte do trabalho que sustenta a IA é Engenharia de Dados.** Modelos sofisticados desabam sobre dados mal encanados.

| Dimensão | Engenharia de Dados | Ciência de Dados |
|---|---|---|
| Pergunta central | "Como entrego este dado confiável, no formato certo, na hora certa?" | "O que este dado me diz? O que consigo prever?" |
| Entregável típico | Pipeline, tabela tratada, API de dados, data warehouse | Modelo, análise, relatório, dashboard |
| Preocupações | Confiabilidade, escala, latência, custo, governança | Acurácia, significância, interpretabilidade |
| Stack recorrente | Python, SQL, Spark, orquestradores, bancos | Python/R, pandas, scikit-learn, notebooks |
| Falha típica | Pipeline quebrou às 3h da manhã | Modelo com viés ou overfitting |

### Onde isso vive no mundo real

Num banco (contexto que conheço de perto), o engenheiro de dados é quem garante que as **transações de ontem** estejam disponíveis, consistentes e auditáveis numa tabela hoje de manhã — para que o time de risco rode seus modelos e a diretoria veja seus indicadores. Se esse dado chega quebrado, atrasado ou sem rastreabilidade, *todo o andar de cima para*.

> **Pontos de Atenção**
> - Engenharia de Dados **não é** "só mexer em banco de dados". É projetar sistemas que tratam dados como um produto vivo.
> - Não confunda com Engenharia de *Software* genérica: aqui o dado é o protagonista, e desafios de **volume, qualidade e tempo** dominam as decisões.
> - A fronteira com Ciência de Dados é porosa e varia por empresa. O que não varia: **sem dado confiável, não há ciência de dado nenhuma.**

> 🔗 **Conexão com a Prática:** seção *1 — Mapa do território* do notebook desenha o ciclo de vida da Engenharia de Dados em código.

---

## 2. O dado como produto: do CSV bruto ao dado confiável

### A ideia central

Um princípio que organiza todo o curso: **o dado é um produto**, não um subproduto. Um produto tem dono, tem padrão de qualidade, tem documentação e tem consumidores que dependem dele. Quando tratamos uma tabela como produto, fazemos perguntas como: *Quem consome? Com que frequência precisa? O que acontece se vier vazia? Como o consumidor sabe que pode confiar?*

### O caminho do dado bruto ao dado confiável

Considere um arquivo `vendas.csv` que chega todo dia de um sistema legado. Ele quase nunca está pronto para uso:

1. **Bruto (raw):** vem como veio da fonte — com datas em formatos inconsistentes, valores faltando, duplicatas, encoding errado. É a "água não tratada".
2. **Limpo (cleaned):** padronizamos tipos, tratamos ausências, removemos duplicatas, validamos regras (ex.: `valor >= 0`).
3. **Modelado (curated):** organizamos em um formato pensado para consumo — tabelas que respondem perguntas de negócio rapidamente.
4. **Servido (served):** disponibilizamos via tabela, API ou arquivo, com **contrato** claro: quais colunas, quais tipos, qual frequência de atualização.

Essa progressão é frequentemente organizada em **camadas** (um padrão popular as chama de *bronze → silver → gold*). Não precisamos do jargão hoje; precisamos da intuição: **dado bruto e dado confiável são coisas diferentes, e a distância entre eles é exatamente o trabalho de Engenharia de Dados.**

### O que torna um dado "confiável"

- **Completo:** o que deveria estar lá, está.
- **Consistente:** o mesmo conceito tem o mesmo formato em todo lugar (uma data é sempre `AAAA-MM-DD`).
- **Pontual (timely):** chega dentro da janela em que ainda é útil.
- **Rastreável (lineage):** consigo responder "de onde veio este número?".

> **Pontos de Atenção**
> - "Garbage in, garbage out": o pipeline mais elegante não conserta uma fonte ruim — só propaga o problema mais rápido.
> - **Validar é mais barato cedo.** Um dado errado pego na ingestão custa minutos; o mesmo erro descoberto num relatório da diretoria custa credibilidade.
> - Documentar o **contrato** do dado (colunas, tipos, significado) é parte do produto, não um extra.

> 🔗 **Conexão com a Prática:** seção *2 — Do bruto ao confiável* do notebook gera um CSV "sujo" de propósito e o transforma em dado limpo, passo a passo.

---

## 3. Por que programar: linguagens no ecossistema de dados

### Por que não basta Excel / cliques

Ferramentas visuais resolvem o caso pequeno e único. A Engenharia de Dados vive do oposto: **processos que repetem todo dia, sobre volumes que não cabem numa planilha, com necessidade de auditoria.** Programar dá três coisas que o clique não dá:

- **Automação:** o processo roda sozinho, todo dia, à mesma hora, sem alguém clicando.
- **Reprodutibilidade:** o mesmo código sobre o mesmo dado dá sempre o mesmo resultado — e isso é auditável.
- **Escala:** o que funciona para mil linhas funciona para milhões com a ferramenta certa.

### O mapa de linguagens (e quando cada uma entra)

Você não precisa dominar todas hoje. Precisa saber **para que serve cada uma**:

- **Python** — a linguagem de cola do ecossistema. Lê de quase tudo, transforma, integra serviços, orquestra. É generalista, legível e tem o maior ecossistema de bibliotecas de dados. **É a linguagem central desta disciplina.**
- **SQL** — a língua franca dos dados estruturados. Você vai usar SQL a vida inteira na área, independentemente da linguagem "principal". Pensar em conjuntos (set-based) é uma habilidade que esta disciplina começa a plantar.
- **Scala / Java** — historicamente o coração de motores de processamento distribuído (o Apache Spark nasceu em Scala; o ecossistema Hadoop, em Java). Aparecem quando se desce ao nível de plataformas de larga escala e alta performance. Não programaremos Scala aqui, mas você precisa saber **por que ela existe na sala**.
- **Outras (Go, Rust, Bash)** — surgem em nichos: serviços de alta performance, ferramentas de linha de comando, automação de infraestrutura.

> **Analogia:** Python é o canivete suíço que você carrega sempre. SQL é a chave que abre o cofre dos dados estruturados. Scala/Java são as máquinas pesadas da fábrica — você não as carrega no bolso, mas sabe que movem o que é grande demais para a mão.

### Linguagem **compilada** vs. **interpretada** (uma distinção que importa)

- **Interpretada (ex.: Python):** o código roda linha a linha, traduzido na hora. Vantagem: rapidez para escrever e testar. Custo: geralmente mais lento em execução pura.
- **Compilada (ex.: Java, Scala, C):** o código é traduzido antecipadamente para uma forma que a máquina executa direto. Vantagem: performance. Custo: ciclo de desenvolvimento mais rígido.

Isso explica a divisão de trabalho do ecossistema: usamos **Python pela produtividade** e, quando o volume aperta, apoiamo-nos em **motores compilados por baixo** (o pandas, por exemplo, faz operações pesadas em C otimizado, não em Python puro).

> **Pontos de Atenção**
> - "Qual a melhor linguagem?" é a pergunta errada. A certa é "**qual a ferramenta certa para este problema?**".
> - Saber SQL **não é opcional** em dados, mesmo que seu dia a dia seja Python.
> - Performance raramente vem de "Python mais rápido"; vem de **usar a estrutura/biblioteca certa** (vetorização, processamento em lote, motor distribuído).

> 🔗 **Conexão com a Prática:** seção *3 — Por que código* do notebook compara fazer "na mão" vs. automatizar, e mostra a mesma pergunta respondida em Python e em SQL (via DuckDB embutido).

---

## 4. Python como linguagem de cola: primeiros passos

### Por que Python virou padrão em dados

Quatro razões objetivas: **legibilidade** (parece pseudocódigo), **ecossistema** (pandas, NumPy, PySpark, requests, e milhares de bibliotecas), **integração** (conecta bancos, APIs, arquivos, nuvem com poucas linhas) e **comunidade** (resposta para quase todo problema já existe). McKinney — criador do pandas — argumenta que essa combinação fez do Python o ambiente padrão de trabalho com dados.

### Os blocos mínimos da linguagem

Para uma turma heterogênea, fixamos só o essencial para ler e escrever um pipeline. Tudo isto é executado no notebook.

**Variáveis** — um nome que guarda um valor:

```python
nome_arquivo = "vendas.csv"
total_linhas = 0
taxa_imposto = 0.18
```

**Tipos básicos** — Python infere o tipo, mas você precisa reconhecê-los:

```python
texto    = "transação"     # str  — texto
inteiro  = 42              # int  — número inteiro
decimal  = 19.90           # float — número com casas decimais
booleano = True            # bool — verdadeiro/falso
vazio    = None            # ausência de valor (importantíssimo em dados!)
```

**Operações e comparações:**

```python
subtotal = 100.0
imposto  = subtotal * taxa_imposto   # 18.0
total    = subtotal + imposto        # 118.0
eh_caro  = total > 100               # True
```

**Condicionais** — decidir o que fazer conforme o dado:

```python
if total > 100:
    categoria = "alto valor"
elif total > 50:
    categoria = "valor médio"
else:
    categoria = "baixo valor"
```

**Laços (loops)** — repetir uma ação sobre vários itens. É o coração de qualquer pipeline, porque pipelines processam *muitas* linhas:

```python
valores = [10.0, 250.0, 75.5, 5.0]
soma = 0.0
for v in valores:
    soma = soma + v
print(soma)   # 340.5
```

**Funções** — empacotar uma lógica com nome, para reutilizar e testar:

```python
def calcular_total(subtotal, taxa):
    """Recebe subtotal e taxa; devolve o total com imposto."""
    return subtotal * (1 + taxa)

calcular_total(100.0, 0.18)   # 118.0
```

> **Conexão com Engenharia de Dados:** repare que cada bloco acima é uma peça de pipeline. *Ler um arquivo* usa variáveis; *limpar* usa condicionais; *processar todas as linhas* usa laços; *padronizar uma regra de negócio* usa funções. Não estamos aprendendo Python "em geral" — estamos aprendendo os blocos que montam fluxos de dados.

> **Pontos de Atenção**
> - **Indentação não é estética em Python — é sintaxe.** O recuo define o bloco. Errar o recuo quebra o código.
> - `None` é o "valor ausente" do Python e vai aparecer o tempo todo em dados reais (campos vazios). Tratá-lo é metade do trabalho de limpeza.
> - Nomes claros (`valor_total` em vez de `x`) são uma decisão de engenharia: pipeline é código que **outra pessoa vai manter** — frequentemente você mesmo, seis meses depois.

> 🔗 **Conexão com a Prática:** seção *4 — Blocos de Python* do notebook executa cada um desses trechos e propõe pequenas variações.

---

## 5. Tipos e estruturas de dados que todo pipeline usa

Variáveis simples não bastam: pipelines movem **coleções** de dados. Quatro estruturas nativas do Python aparecem em praticamente todo fluxo.

### Lista — sequência ordenada, mutável

Uma fila de itens na ordem em que chegaram. É a estrutura mais comum para representar "várias linhas", "vários arquivos", "vários registros".

```python
arquivos = ["jan.csv", "fev.csv", "mar.csv"]
arquivos.append("abr.csv")        # adiciona ao fim
primeiro = arquivos[0]            # acesso por índice (começa em 0!)
print(len(arquivos))              # 4 — quantos itens
```

**Quando usar:** ordem importa, há repetição, você vai percorrer item a item.

### Dicionário — pares chave→valor

A estrutura mais importante para **representar um registro**. Pense numa linha de tabela: cada coluna é uma chave, cada célula é um valor. É também a forma natural do **JSON**, formato onipresente em APIs.

```python
transacao = {
    "id": 1001,
    "valor": 250.00,
    "categoria": "eletronicos",
    "aprovada": True,
}
transacao["valor"]                # 250.0 — acesso por chave
transacao["device"] = "mobile"    # adiciona campo
```

**Quando usar:** dados com nome (registros, configurações, respostas de API). Acesso por chave é rápido e legível.

### Tupla — sequência ordenada, **imutável**

Como uma lista, mas que não pode ser alterada depois de criada. Útil para representar algo que não deve mudar — uma coordenada, um par fixo, uma chave composta.

```python
coordenada = (-3.7327, -38.5267)   # Fortaleza; não faz sentido "mudar pela metade"
```

### Conjunto (set) — coleção sem ordem e **sem duplicatas**

Resolve de graça um problema clássico de dados: **eliminar repetições** e testar pertinência rapidamente.

```python
categorias = {"alimentacao", "transporte", "alimentacao"}
print(categorias)                 # {'alimentacao', 'transporte'} — duplicata sumiu
"transporte" in categorias        # True — teste de pertinência instantâneo
```

### Por que essas quatro importam tanto

| Estrutura | Pergunta que responde | No mundo dos dados |
|---|---|---|
| Lista | "Quais itens, nesta ordem?" | Linhas de um arquivo, lote de registros |
| Dicionário | "Quais campos tem este registro?" | Uma linha de tabela, um objeto JSON de API |
| Tupla | "Qual este par/conjunto fixo?" | Coordenada, chave composta, registro imutável |
| Conjunto | "Quais valores únicos existem?" | Deduplicação, categorias distintas, validação |

Quando passarmos ao **pandas** nas próximas aulas, o `DataFrame` (a "tabela" do Python) é, por baixo, uma combinação sofisticada dessas ideias — colunas nomeadas (como dicionário) de sequências ordenadas (como listas). Dominar as estruturas nativas hoje é o que torna o pandas intuitivo depois.

> **Pontos de Atenção**
> - Índice de lista **começa em 0**, não em 1. Fonte clássica de erro para quem vem de planilhas.
> - Listas são mutáveis; tuplas não. Escolher a estrutura certa **comunica intenção** a quem lê o código.
> - Dicionário e JSON são quase a mesma forma mental — isso vai facilitar muito quando consumirmos APIs.

> 🔗 **Conexão com a Prática:** seção *5 — Estruturas de dados* do notebook manipula as quatro e mostra como uma lista de dicionários já é, essencialmente, uma tabela.

---

## 6. Primeiro mini-pipeline: ler, transformar e salvar

### O objetivo do encontro converge aqui

Tudo que vimos serve a um propósito: construir, ainda que em miniatura, **o padrão que se repete em toda a Engenharia de Dados** — frequentemente chamado de **ETL** (*Extract, Transform, Load*) ou, em variações modernas, **ELT**:

```
EXTRAIR (ler a fonte) → TRANSFORMAR (limpar/enriquecer) → CARREGAR (salvar onde será consumido)
```

### O pipeline mínimo, em pseudocódigo

```
1. EXTRAIR:   ler vendas.csv para a memória
2. TRANSFORMAR:
     - padronizar a data para AAAA-MM-DD
     - remover linhas com valor ausente ou negativo
     - calcular coluna "total_com_imposto"
3. CARREGAR:  salvar vendas_tratadas.csv (e/ou em um banco)
```

### Por que isso é "engenharia", e não só "um script"

Um script roda uma vez. Um **pipeline** é pensado para rodar **muitas vezes, de forma confiável**. A diferença está nas perguntas que fazemos ao redor do código:

- **Idempotência:** se rodar duas vezes, o resultado é o mesmo? (Não pode duplicar dados.)
- **Tratamento de erro:** se a fonte vier vazia ou malformada, o pipeline **falha de forma clara** ou corrompe silenciosamente?
- **Rastreabilidade:** consigo saber quantas linhas entraram, quantas foram descartadas e por quê?
- **Observabilidade:** o pipeline **conta o que fez** (logs), para que eu confie nele sem assistir rodar?

Não vamos implementar tudo isso hoje em profundidade — vários desses temas são encontros inteiros adiante. Mas **plantamos a mentalidade desde a primeira aula**: código de dados que vai para produção carrega responsabilidades que um script de uso único não tem.

### O mesmo pipeline cresce com você

O mini-pipeline de hoje cabe em 20 linhas de Python puro. Nas próximas semanas, a mesma estrutura conceitual ganha:

- **pandas**, para transformar tabelas com poucas linhas em vez de laços manuais;
- **SQL**, para deixar o banco fazer o trabalho pesado;
- **validação de qualidade**, para garantir contratos;
- **orquestração**, para rodar sozinho na hora certa e avisar quando falha.

O fio condutor não muda: **extrair, transformar, carregar — com confiabilidade.**

> **Pontos de Atenção**
> - "Funcionou na minha máquina" não é critério de pipeline. Reprodutibilidade e tratamento de erro são parte do entregável.
> - Comece simples e correto; só então otimize. Pipeline rápido que entrega dado errado é pior que pipeline lento e correto.
> - Todo descarte de dado deve ser **contado e justificado** — silêncio é a origem de erros que aparecem semanas depois.

> 🔗 **Conexão com a Prática:** seção *6 — Seu primeiro pipeline* do notebook executa o ETL completo de ponta a ponta sobre um CSV sintético, contando linhas mantidas e descartadas.

---

## Síntese do encontro

Hoje construímos o alicerce conceitual do curso:

1. **Engenharia de Dados** constrói o encanamento confiável que alimenta análise, IA e negócio — é distinta de, e pré-requisito para, Ciência de Dados.
2. **O dado é um produto**: existe uma distância real entre dado bruto e dado confiável, e fechá-la é o nosso trabalho.
3. **Programar** dá automação, reprodutibilidade e escala — coisas que o clique manual não entrega.
4. **Python** é a linguagem de cola central; **SQL** é inegociável; **Scala/Java** explicam o motor de larga escala.
5. As **estruturas de dados nativas** (lista, dicionário, tupla, conjunto) são as peças com que montamos fluxos — e a base do pandas que virá.
6. O padrão **Extrair → Transformar → Carregar** é o esqueleto que vamos repetir, refinar e escalar ao longo de toda a disciplina.

### Para o próximo encontro

- Rode o notebook inteiro por conta própria; altere valores e observe o que muda.
- Pense em **um processo de dados do seu próprio trabalho** (uma planilha que você atualiza todo mês, um relatório que você monta na mão). Traga-o: vamos enxergá-lo como um pipeline.

---

## Bibliografia

- REIS, Joe; HOUSLEY, Matt. *Fundamentos de Engenharia de Dados*. O'Reilly / Alta Books, 2023.
- MCKINNEY, Wes. *Python para Análise de Dados*. 3ª ed. O'Reilly, 2022.
- KLEPPMANN, Martin. *Designing Data-Intensive Applications*. O'Reilly, 2017.

---

*Material de apoio — UNIFOR Pós-Graduação · Prof. Cassio Pinheiro. Acompanha o notebook prático e os slides da disciplina.*
