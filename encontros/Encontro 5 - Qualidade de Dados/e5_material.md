# Linguagens de Programação para Engenharia de Dados
## Encontro 5 — Qualidade e confiabilidade de dados

**Instituição:** Universidade de Fortaleza (UNIFOR)
**Curso:** Pós-Graduação Lato Sensu — Especialização em Engenharia de Dados
**Professor:** Cassio Pinheiro · [LinkedIn](https://www.linkedin.com/in/cassio-pinheiro-9baa7939/)
**Data:** 10/07/2026 · 19:00 às 22:30
**Carga horária da disciplina:** 24h

---

## Como ler este documento

Este é o material de fundamentação do **quinto encontro**. Ele acompanha:

- o **notebook** `e5_pratica.ipynb` (onde rodamos código — cada seção do notebook é referenciada aqui com a marca 🔗);
- os **slides** `e5_slides.pptx` (para a aula expositiva).

Nos encontros anteriores construímos o vocabulário (E1), domamos o pandas (E2), exercitamos transformação e SQL (E3) e enfrentamos a escala (E4). Em todos eles assumimos, silenciosamente, uma premissa perigosa: **a de que o dado que entrava estava correto**. Hoje atacamos exatamente essa premissa. No E1 plantamos duas ideias que agora amadurecem: a de que **o dado é um produto** (com dono, padrão de qualidade e consumidores) e a de que esse produto tem um **contrato de saída**. O Encontro 5 transforma essas ideias em **código que valida, mede, separa e vigia** — porque um produto sem controle de qualidade não é produto, é risco.

O fio condutor do curso continua sendo o nosso pipeline de vendas. Hoje ele ganha uma camada nova e decisiva: **validação e quarentena**. Em vez de confiar que o dado tratado está bom, vamos *provar* que está — e, quando não estiver, vamos **separar o ruim do bom de forma rastreável**, em vez de deixá-lo contaminar o consumidor.

> **Combinado pedagógico:** qualidade de dados não é "rodar uma checagem no fim". É um contrato que se afirma cedo, se mede continuamente e se vigia sozinho. O bom engenheiro descobre que o dado quebrou **antes** do consumidor — não pelo telefonema do diretor.

---

## 1. Dimensões de qualidade: do conceito à medição

### Por que "dado bom" precisa de definição operacional

No E1 dissemos que um dado confiável é completo, consistente, pontual e rastreável. Aquilo era intuição; hoje é **engenharia**. "Qualidade" é vaga demais para virar código — precisamos decompô-la em **dimensões mensuráveis**, cada uma com uma fórmula. Só o que se mede se gerencia, e só o que vira número entra num pipeline automatizado.

As seis dimensões clássicas, e como medir cada uma sobre uma tabela:

| Dimensão | Pergunta | Como medir |
|---|---|---|
| **Completude** | O que deveria estar lá, está? | % de células não-nulas por coluna; % de linhas sem campos obrigatórios vazios |
| **Validade** | O valor obedece à regra/formato? | % de linhas que passam num *check* (regex, range, domínio) — ex.: `valor >= 0` |
| **Consistência** | O mesmo conceito tem o mesmo formato em todo lugar? | datas sempre `AAAA-MM-DD`; categoria sempre num conjunto fechado; coerência entre colunas |
| **Unicidade** | Cada entidade aparece uma vez só? | nº de duplicatas na chave primária (`id`); cardinalidade esperada |
| **Pontualidade** (timeliness) | O dado chegou na janela em que ainda é útil? | *lag* entre o evento e sua disponibilidade; idade do dado mais recente vs. SLA |
| **Acurácia** | O valor corresponde ao mundo real? | conferência contra fonte de verdade (golden source), amostragem, reconciliação |

A acurácia é a mais difícil das seis: as outras cinco se medem **olhando só para a tabela**; acurácia exige uma **referência externa** ("o total de vendas bate com o que o ERP fechou?"). Por isso, na prática, automatizamos pesadamente as cinco primeiras e tratamos acurácia com reconciliações periódicas.

### Qualidade como atributo do produto

Voltando ao E1: se o dado é um **produto**, essas dimensões são a sua **ficha de especificação** — o equivalente à etiqueta de um produto físico ("composição", "validade", "lote"). Um consumidor que recebe a tabela tem o direito de saber: *qual a completude desta coluna? a chave é mesmo única? quando foi a última atualização?* Quando publicamos esses números junto com o dado, paramos de pedir confiança e passamos a **oferecer evidência**.

> **Pontos de Atenção**
> - Não existe "qualidade 100%" como meta universal. Cada dimensão tem um **limiar aceitável** definido pelo consumidor (ex.: "completude de `cliente_id` ≥ 99,5%"). Qualidade é contrato, não perfeição.
> - Meça **por coluna**, não só por tabela. Uma tabela "98% completa" pode esconder uma coluna crítica 40% vazia.
> - Dimensões podem conflitar: aumentar **pontualidade** (entregar mais cedo) às vezes reduz **completude** (dados que chegam atrasados ainda não vieram). A decisão é de negócio.

> 🔗 **Conexão com a Prática:** seção *1 — Medindo qualidade* do notebook calcula completude por coluna, duplicatas e taxa de validade sobre o vendas tratado.

---

## 2. Validação programática com Pandera

### O salto: da checagem ad-hoc ao schema declarativo

Até aqui, quando queríamos validar, escrevíamos `if`s soltos: `df[df.valor < 0]`. Funciona para um caso, mas não escala — vira um emaranhado de checagens dispersas, sem documentação e impossível de reusar. **Pandera** resolve isso deixando você *declarar* o que uma tabela válida deve ser, num único objeto, e validar contra ele.

A peça central é o **`DataFrameSchema`**: um dicionário de colunas, cada uma um **`Column`** com tipo, restrições de nulo, e uma lista de **`Check`** (regras de valor). Validar é uma chamada: `schema.validate(df)`.

```python
import pandera.pandas as pa

schema = pa.DataFrameSchema({
    "id":        pa.Column(int,   pa.Check.greater_than(0), unique=True),
    "valor":     pa.Column(float, pa.Check.greater_than_or_equal_to(0)),
    "data":      pa.Column(object, pa.Check.str_matches(r"^\d{4}-\d{2}-\d{2}$")),
    "categoria": pa.Column(object, pa.Check.isin(["eletronicos", "alimentacao", "transporte"])),
})

validado = schema.validate(df)   # devolve o df se válido; levanta SchemaError se não
```

Repare como cada linha do schema é, ao mesmo tempo, **regra executável e documentação**. O schema *é* a especificação do produto — e ela não pode ficar desatualizada, porque é ela que roda.

### Checks: o vocabulário de regras

Pandera traz *checks* prontos — `greater_than_or_equal_to`, `isin`, `str_matches`, `in_range` — e permite **checks customizados** com qualquer função booleana:

```python
# regra de negócio: nenhuma venda futura
pa.Check(lambda s: pd.to_datetime(s) <= pd.Timestamp.today(), error="data no futuro")
```

### Validação preguiçosa: ver *todos* os erros de uma vez

Por padrão, Pandera para no primeiro erro. Em diagnóstico isso é ruim: queremos o **relatório completo**. Com `lazy=True`, ele coleta **todas** as falhas e as levanta juntas num `SchemaErrors` (plural), cuja propriedade `.failure_cases` é um DataFrame com coluna, check violado e o valor exato que falhou — ouro puro para a quarentena da Seção 5.

```python
try:
    schema.validate(df, lazy=True)
except pa.errors.SchemaErrors as e:
    print(e.failure_cases)   # tabela de TODAS as violações
```

> **Pontos de Atenção**
> - A partir da versão 0.20, o import correto para pandas é `import pandera.pandas as pa` — o antigo `import pandera as pa` ainda funciona mas está em depreciação. Confira a sua versão.
> - `coerce=True` numa coluna faz Pandera **converter** o tipo antes de validar (ex.: string `"10"` vira `int`). Útil, mas use com consciência: conversão silenciosa pode mascarar sujeira.
> - Um schema é **código versionado**. Trate-o como qualquer artefato: revisão, testes, histórico. Quando o contrato muda, o schema muda — e o diff conta a história.

> 🔗 **Conexão com a Prática:** seção *2 — Validação com Pandera* do notebook define o schema, valida com sucesso e força uma falha proposital capturada com `lazy=True`.

---

## 3. Great Expectations: validação como suíte e documentação viva

### O que Pandera não cobre sozinho

Pandera é leve e perfeito para validar dentro do código de um pipeline. **Great Expectations (GE)** ataca um problema vizinho e maior: tratar validação como uma **suíte de expectativas** versionada, com **documentação automática navegável** (os *Data Docs*) e **pontos de verificação** (*checkpoints*) que se plugam ao orquestrador. É a diferença entre uma função de validação e uma *plataforma* de qualidade.

Os três conceitos centrais:

- **Expectation Suite** — uma coleção de *expectativas* sobre um dataset, em linguagem quase natural: `expect_column_values_to_not_be_null("cliente_id")`, `expect_column_values_to_be_between("valor", min_value=0)`, `expect_column_values_to_be_in_set("categoria", [...])`. Cada expectativa é declarativa e auto-descritiva.
- **Checkpoint** — a unidade que **executa** uma suíte contra um lote de dados numa hora definida (tipicamente disparado pelo orquestrador, tema do E6) e decide o que fazer com o resultado: passar, alertar, ou bloquear o pipeline.
- **Data Docs** — páginas HTML geradas automaticamente a partir das execuções: o que se esperava, o que passou, o que falhou, com exemplos. É a **documentação que não mente**, porque nasce da própria validação.

```python
# esboço conceitual (a API exata varia por versão de GE)
import great_expectations as gx

context = gx.get_context()
validator = context.sources.pandas_default.read_dataframe(df)
validator.expect_column_values_to_be_between("valor", min_value=0)
validator.expect_column_values_to_be_in_set("categoria",
        ["eletronicos", "alimentacao", "transporte"])
resultado = validator.validate()      # alimenta os Data Docs
```

### Quando GE, quando Pandera

Pandera ganha quando você quer validação **embutida e leve** no código Python, com baixo atrito. GE ganha quando a organização precisa de **governança**: catálogo de expectativas compartilhado entre times, documentação navegável para auditoria e integração formal com o orquestrador. Não são rivais — muitos times usam Pandera no *runtime* e GE como camada de governança e *reporting*.

> **Pontos de Atenção**
> - GE é **pesado e tem API instável entre versões maiores** (a forma de criar contexto e checkpoints mudou bastante de 0.15 para 1.x). No notebook, por isso, GE é tratado em `try/except`: se não estiver instalado ou a versão divergir, mostramos o código equivalente e seguimos. **Nunca** faça o pipeline depender de uma biblioteca instável para *rodar*.
> - Não confunda "muitas expectativas" com "boa qualidade". Expectativa que ninguém entende ou que nunca falha vira ruído. Cada regra deve proteger um consumidor real.

> 🔗 **Conexão com a Prática:** seção *3 — Great Expectations* do notebook tenta usar GE; se indisponível, imprime o equivalente e explica suites, checkpoints e Data Docs.

---

## 4. Data contracts: o contrato como código entre produtor e consumidor

### A formalização do "contrato de saída" do E1

No E1, ao falar de dado servido, dissemos que ele vai ao consumidor "com **contrato** claro: quais colunas, quais tipos, qual frequência". Aquele contrato era informal — um combinado verbal ou um documento que envelhece. Um **data contract** é esse combinado **virado código**: um artefato versionado, acordado entre **produtor** (quem gera o dado) e **consumidor** (quem depende dele), que o pipeline pode ler e *fazer cumprir automaticamente*.

Um contrato de dados completo declara três camadas:

1. **Schema** — colunas, tipos, nulabilidade, chave primária, domínios permitidos. (É o que um `DataFrameSchema` Pandera já expressa.)
2. **SLAs** — garantias operacionais: frequência de atualização ("diária, até 6h"), pontualidade, completude mínima por coluna, política de versionamento.
3. **Semântica** — o *significado* de cada campo: o que é `valor` (bruto? líquido? em qual moeda?), como `categoria` é definida, quais regras de negócio se aplicam. É a parte mais esquecida e a que mais causa incidentes.

### Por que isso é tendência forte de mercado

A dor que os contratos resolvem é real e cara: o produtor muda uma coluna ("vamos renomear `valor` para `valor_bruto`") sem avisar, e **três dashboards quebram silenciosamente** rio abaixo. O contrato inverte a lógica: a mudança que viola o acordo **falha na origem**, no CI do produtor, e não na cara do consumidor. Isso desloca a responsabilidade para quem causa o problema — o princípio de **shift-left** (Seção 6) aplicado à organização.

Na prática, um contrato costuma ser um arquivo declarativo (YAML/JSON) que **gera** o schema de validação. O mesmo arquivo serve a três públicos: o produtor (que se compromete), o consumidor (que sabe o que esperar) e o pipeline (que valida).

> **Pontos de Atenção**
> - Contrato sem *enforcement* é só documentação — e documentação apodrece. O valor aparece quando ele **roda no CI/CD** e quebra o build de quem o viola.
> - Versione o contrato e trate mudanças como mudanças de API: *breaking changes* (remover coluna, mudar tipo) exigem aviso e janela de migração; mudanças aditivas são seguras.
> - O contrato é tão bom quanto a sua camada **semântica**. Schema certo com significado ambíguo ainda gera o relatório errado.

> 🔗 **Conexão com a Prática:** seção *4 — Data contract* do notebook expressa um contrato como dicionário e o materializa num schema Pandera executável.

---

## 5. Estratégias de tratamento: fail-fast, tolerância e quarentena

### A pergunta que define a arquitetura: o que fazer com o dado ruim?

Detectar é metade do trabalho; a outra metade é **decidir o que fazer com o que falhou**. Há dois extremos e um meio-termo que é, quase sempre, a resposta certa em dados.

- **Fail-fast (parar tudo):** ao primeiro erro, o pipeline **aborta** e não entrega nada. Vantagem: nenhum dado ruim chega ao consumidor. Custo: um único registro podre derruba o lote inteiro — inaceitável quando 999.999 de 1.000.000 de linhas estão perfeitas. Apropriado quando a integridade é absoluta (ex.: razão contábil).
- **Tolerante (deixar passar):** processa tudo, ignorando ou "consertando" silenciosamente o que está errado. Vantagem: nada para. Custo: você propaga sujeira e perde rastreabilidade — o pior dos mundos para confiabilidade.
- **Quarentena (a via do meio):** **separa** as linhas válidas das inválidas. As válidas seguem para o consumidor; as inválidas vão para uma área isolada, **enriquecidas com o motivo da falha**, para investigação e reprocessamento. O lote entrega o que é bom **e** não perde o que é ruim.

### Quarentena na prática

O padrão tem quatro passos:

1. **Validar com `lazy=True`** para coletar todas as falhas de uma vez.
2. **Particionar:** os índices que aparecem em `failure_cases` formam o conjunto "inválido"; o complemento é o "válido".
3. **Enriquecer** a tabela de quarentena com colunas de diagnóstico: qual coluna falhou, qual check, qual valor — para que ninguém precise adivinhar.
4. **Persistir os dois fluxos** separadamente (`vendas_validas` e `vendas_quarentena`) e **contar** ambos. Esse número (taxa de quarentena) é, em si, uma métrica de qualidade que se vigia ao longo do tempo (Seção 6).

O **reprocessamento** fecha o ciclo: linhas em quarentena não são lixo, são trabalho pendente. Corrigida a fonte (ou a regra), o lote em quarentena pode ser **reinjetado** no pipeline. Por isso a quarentena guarda o registro original *e* o motivo — sem o motivo, não há como saber o que corrigir.

> **Pontos de Atenção**
> - **Todo descarte tem que ser contado e justificado** (eco direto do E1). Linha que some sem registro é a origem de erros que aparecem semanas depois — "por que o faturamento do mês veio menor?".
> - Quarentena não é solução permanente: é uma sala de espera. Se a taxa de quarentena cresce e ninguém reprocessa, o problema só mudou de lugar.
> - Cuidado com o "conserto silencioso" (imputar média, truncar, preencher com zero). Às vezes é legítimo, mas **sempre** registre que houve conserto — senão você fabricou dado sem rastro.

> 🔗 **Conexão com a Prática:** seção *5 — Quarentena* do notebook separa válidas de inválidas, anexa o motivo do erro e conta os dois fluxos.

---

## 6. Observabilidade e lineage: saber que quebrou antes do consumidor

### Validar é um instante; observar é ao longo do tempo

A validação da Seção 2 responde "este lote, agora, está válido?". A **observabilidade de dados** responde a uma pergunta mais difícil e mais valiosa: "este pipeline está **saudável ao longo do tempo**?". A diferença é a mesma entre tirar a temperatura uma vez e ter um monitor cardíaco. Em dados, observabilidade se apoia em quatro sinais (os "pilares"):

- **Volume (freshness/volume):** o lote de hoje tem mais ou menos linhas do que o normal? Chegou na hora?
- **Distribuição:** os valores estão dentro da faixa histórica? Uma coluna `valor` cuja média triplica de um dia para o outro é um sinal — mesmo que cada linha seja "válida".
- **Schema:** uma coluna sumiu, mudou de tipo ou apareceu nova? (É a violação de contrato da Seção 4, detectada em produção.)
- **Lineage:** de onde este dado veio e o que ele alimenta?

### Detecção de anomalias e métricas no tempo

A ideia-chave é gravar as **métricas de qualidade de cada execução** (completude por coluna, taxa de quarentena, contagem de linhas, médias) numa série temporal. Com o histórico, "anomalia" deixa de ser um limiar fixo e vira **desvio do padrão da própria série** — completude que cai de 99,8% para 91%, volume que despenca pela metade. O sistema **alerta o engenheiro**, não o consumidor.

### Lineage: o "de onde veio este número?"

**Lineage** (linhagem) é o mapa de dependências: qual fonte alimentou qual transformação, que gerou qual tabela, consumida por qual dashboard. Seu valor é duplo. Para **diagnóstico**: quando uma tabela quebra, o lineage mostra *o que mais será afetado* rio abaixo. Para **impacto de mudança**: antes de alterar a fonte, o lineage mostra *quem depende dela* — exatamente o que um data contract protege. Junto com os **metadados** (descrições, donos, schemas, métricas), o lineage forma a espinha dorsal da governança moderna.

### Caixa de tendência: para onde a qualidade de dados está indo

- **Shift-left data quality.** Empurrar a validação para o **mais cedo possível** no fluxo — idealmente para o CI do produtor, antes de o dado existir em produção. Quanto mais à esquerda você pega o erro, mais barato ele é (eco direto do E1: "validar é mais barato cedo"). É a filosofia que une contratos, testes de schema e validação na ingestão.
- **Data contracts** como interface de primeira classe entre times, executados no pipeline — não mais combinados informais.
- **Ferramentas de observabilidade.** **Soda** (checks de qualidade declarativos em YAML, *open source* e leve), **Great Expectations** (suítes e Data Docs) e **Monte Carlo** (observabilidade gerenciada com detecção de anomalias por ML e lineage automático) formam o espectro do mercado — do leve e embutido ao gerenciado e completo. Conceito a reter: **monitorar o dado virou tão padrão quanto monitorar o servidor.**

> **Pontos de Atenção**
> - Observabilidade não substitui validação — **complementa**. Validação barra o erro conhecido; observabilidade flagra o erro que você não previu.
> - Alerta demais é tão ruim quanto alerta de menos: *alert fatigue* faz a equipe ignorar tudo. Calibre limiares com o histórico, não no chute.
> - Lineage manual envelhece e mente. Prefira lineage **derivado do código** (do próprio SQL/pipeline) sempre que possível.

> 🔗 **Conexão com a Prática:** seção *6 — Observabilidade* do notebook consolida as métricas de qualidade do lote num "cartão de saúde" pronto para virar série temporal.

---

## Síntese do encontro

Hoje demos ao pipeline de vendas a camada que faltava para ele ser **produto** de verdade:

1. **Dimensões de qualidade** — completude, validade, consistência, unicidade, pontualidade e acurácia: cada uma com uma forma de medir, transformando "dado bom" de intuição em número.
2. **Pandera** — validação declarativa com `DataFrameSchema`, `Column` e `Check`; `lazy=True` para o relatório completo de falhas.
3. **Great Expectations** — validação como suíte versionada, com checkpoints e Data Docs; poderosa, porém pesada — por isso tratada com cautela e *fallback*.
4. **Data contracts** — o contrato de saída do E1 virado código (schema + SLAs + semântica), executado no pipeline; tendência forte que desloca a falha para a origem.
5. **Estratégias de tratamento** — fail-fast vs. tolerante, e a **quarentena** como via do meio: separar o ruim do bom, enriquecido com o motivo, sem perder rastro.
6. **Observabilidade e lineage** — métricas ao longo do tempo, detecção de anomalias e mapa de dependências: descobrir que o dado quebrou **antes** do consumidor.

O elo que costura tudo é o **shift-left**: validar cedo é barato, descobrir tarde custa credibilidade.

### Para o próximo encontro

- Rode o notebook inteiro; force novas falhas (mude um valor para negativo, uma categoria para fora do conjunto) e observe a quarentena capturando-as.
- Pense num dado do seu trabalho e escreva, em uma frase, **três cláusulas de contrato** para ele (uma de schema, uma de SLA, uma de semântica). Traga-as.
- No **Encontro 6 — Orquestração e automação de pipelines**, vamos fazer tudo isto rodar **sozinho, na hora certa, avisando quando falha**: é onde os *checkpoints* de validação e as métricas de qualidade encontram o agendador.

---

## Bibliografia

- MCKINNEY, Wes. *Python para Análise de Dados*. 3ª ed. O'Reilly, 2022.
- REIS, Joe; HOUSLEY, Matt. *Fundamentos de Engenharia de Dados*. O'Reilly / Alta Books, 2023.
- KLEPPMANN, Martin. *Designing Data-Intensive Applications*. O'Reilly, 2017.

---

*Material de apoio — UNIFOR Pós-Graduação · Prof. Cassio Pinheiro. Acompanha o notebook prático e os slides da disciplina.*
