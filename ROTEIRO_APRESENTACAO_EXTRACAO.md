# Roteiro de Apresentação — Extração de Dados do YouTube
### Pipeline de Engenharia de Dados sobre transcrições de vídeos

---

## Slide 1 — Capa
**Título:** Extração de Dados do YouTube para Engenharia de Dados
**Subtítulo:** Como transformar legendas de vídeos em dados analíticos confiáveis
- Disciplina: Linguagens de Programação para Engenharia de Dados — UNIFOR
- Projeto: 10 pipelines reais com a `youtube-transcript-api`

**Fala:** "Hoje vou mostrar como extraímos dados do YouTube de forma técnica e resiliente — não os vídeos em si, mas as **transcrições** (legendas), que são uma fonte riquíssima de texto para análise."

---

## Slide 2 — O problema: por que extrair transcrição e não o vídeo?
- O vídeo é pesado, opaco e caro de processar; a **transcrição é texto estruturado e leve**.
- Cada trecho de legenda vem com 3 campos-chave: `text`, `start` (segundo em que começa) e `duration`.
- Esse `start` é ouro: permite analisar **quando** algo é dito, não só **o quê** — densidade por minuto, janelas de retenção, evolução temporal.

**Fala:** "A grande sacada é que a legenda já vem com marcação de tempo. Isso nos deixa fazer análise temporal sem nenhum processamento de áudio."

---

## Slide 3 — Arquitetura: onde a extração se encaixa
Lakehouse local em 3 camadas (Medallion):

| Camada | Papel | Foco de hoje |
|---|---|---|
| **Bronze** | Extração bruta das transcrições | ⭐ **AQUI** |
| Silver | Limpeza + contrato de dados (Pandera) | contexto |
| Gold | Análise por domínio (DuckDB/Polars) | contexto |

**Fala:** "A extração é a camada **Bronze**. Tudo que vem depois depende de ela ser robusta. Se a Bronze falha, o pipeline inteiro para."

---

## Slide 4 — A ferramenta: `youtube-transcript-api` v1.x
- Biblioteca Python que acessa as legendas públicas de um vídeo sem precisar de chave de API do Google.
- **Detalhe técnico importante:** a v1.x mudou a API.
  - ❌ Antigo (estático): `YouTubeTranscriptApi.get_transcript(id)`
  - ✅ Atual (instância): `YouTubeTranscriptApi().fetch(id, languages=...)`

**Fala:** "Esse é um erro clássico que trava muita gente: tutoriais antigos usam o método estático que não existe mais. A versão atual instancia a classe e chama `.fetch()`."

---

## Slide 5 — O que é o ID do vídeo
- A extração é por **ID do vídeo**, não pela URL inteira.
- `youtube.com/watch?v=` **`dQw4w9WgXcQ`** → o ID é o trecho após `v=`.
- No código: a lista `VIDEO_IDS = [...]` é o ponto de entrada do pipeline.

**Fala:** "O usuário cola só o ID. Se a lista ficar vazia, vamos ver que o pipeline ainda assim roda — graças ao fallback."

---

## Slide 6 — Núcleo da extração: `extrair_transcricao()`
```python
def extrair_transcricao(video_id):
    try:
        from youtube_transcript_api import YouTubeTranscriptApi
        api = YouTubeTranscriptApi()              # API v1.x (instância)
        fetched = api.fetch(video_id, languages=IDIOMAS)
        return [{"text": s.text, "start": s.start, "duration": s.duration}
                for s in fetched]
    except Exception as e:
        print(f"  [aviso] {video_id}: legenda indisponível ...")
        return None
```
**3 pontos técnicos para destacar:**
1. **Multilíngue:** `IDIOMAS = ["pt", "pt-BR", "en"]` — tenta na ordem; pega a primeira disponível.
2. **Normalização:** transforma o objeto da API numa lista de dicionários simples (`text/start/duration`).
3. **Resiliência:** qualquer falha → retorna `None`, **não quebra** o programa.

**Fala:** "Repare no `try/except`. Vídeo sem legenda, legenda desativada, problema de rede — tudo cai no `except` e vira `None`, sinalizando para o pipeline buscar uma alternativa."

---

## Slide 7 — O segredo da robustez: fallback sintético
```python
def gerar_sintetico(video_id, n_trechos=180):
    rng = random.Random(hash(video_id) & 0xFFFFFFFF)  # determinístico
    ...
    if rng.random() < 0.30:
        palavras.append(rng.choice(VOCABULARIO))  # injeta termos do domínio
```
- Se não há ID válido **ou** a legenda não existe, o código **gera dados sintéticos**.
- **Determinístico** (`seed=42` + `hash` do id): a mesma entrada gera sempre a mesma saída → a aula roda igual em qualquer máquina, com ou sem internet.
- Mantém a **mesma estrutura** (`text/start/duration`), então as camadas seguintes não percebem diferença.

**Fala:** "Esse é o diferencial de engenharia: o pipeline **nunca para**. Sem legenda disponível, ele produz dados realistas e deterministas. É a diferença entre um script de tutorial e um pipeline de produção."

---

## Slide 8 — Orquestração: `ingerir()`
```python
def ingerir(video_ids):
    registros, origem_sintetica = [], False
    alvos = video_ids if video_ids else [f"demo_{i:02d}" for i in range(3)]
    for vid in alvos:
        trechos = extrair_transcricao(vid) if video_ids else None
        if trechos is None:                       # fallback automático
            trechos = gerar_sintetico(vid)
            origem_sintetica = True
        for ordem, tr in enumerate(trechos):
            registros.append({"video_id": vid, "ordem": ordem, ...})
    df = pd.DataFrame(registros)
```
- Decide entre **real vs. sintético** por vídeo.
- Adiciona a coluna **`ordem`** (sequência do trecho) → preserva a ordem da fala.
- Consolida tudo num **DataFrame pandas** (a saída da Bronze).
- A flag `origem_sintetica` avisa o usuário da procedência dos dados (rastreabilidade).

**Fala:** "Aqui está a lógica de decisão. Note a coluna `ordem`: ela é a chave que mantém a sequência temporal da transcrição, junto com o `start`."

---

## Slide 9 — Saída da Bronze: o contrato bruto
Cada linha extraída = um trecho de fala:

| video_id | ordem | texto | start | duration |
|---|---|---|---|---|
| abc123 | 0 | "o ponto principal aqui" | 0.0 | 3.2 |
| abc123 | 1 | "isso significa juros" | 3.2 | 4.1 |

- Salvo em **Parquet** em `./datalake/bronze/` (formato colunar, eficiente).
- É a matéria-prima que a Silver vai limpar e validar.

**Fala:** "A Bronze entrega dado **bruto mas estruturado**. Não limpamos nada ainda — princípio do Medallion: a Bronze preserva o dado como veio, para ser auditável."

---

## Slide 10 — Decisões de engenharia que importam
Resumo do que torna esta extração "de produção":
1. **Idempotência / determinismo** — `seed=42`, roda igual sempre.
2. **Tolerância a falhas** — `try/except` por vídeo, nunca derruba o lote.
3. **Degradação graciosa** — fallback sintético em vez de erro fatal.
4. **Normalização na entrada** — estrutura única independente da origem.
5. **Rastreabilidade** — flag de origem + camada Bronze imutável.

**Fala:** "Esses cinco princípios são o que separa extrair dados 'no improviso' de construir um pipeline confiável."

---

## Slide 11 — Como rodar (demo)
1. Copiar 2-3 IDs de vídeos reais do domínio e colar em `VIDEO_IDS`.
2. Executar a célula Bronze → ver as transcrições extraídas.
3. (Opcional) Deixar a lista vazia → ver o fallback sintético entrar em ação.

**Fala (demo ao vivo):** "Vou rodar com a lista vazia primeiro para mostrar o fallback, e depois com um ID real para mostrar a extração de verdade."

---

## Slide 12 — Fechamento
- Extraímos **texto temporalizado** do YouTube sem chave de API.
- Construímos uma camada Bronze **resiliente, determinista e rastreável**.
- Essa base alimenta análises Gold por domínio (densidade de termos, sentimento, retenção...).

**Frase de efeito:** "Bons dados não começam na análise — começam numa extração que nunca te deixa na mão."
