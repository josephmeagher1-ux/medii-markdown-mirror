# Architecture options — alternatives at every stage of the Medii pipeline

> **Status:** research deliverable, 2026-04-26. Operator-requested.
> **Scope:** enumerates options + trade-offs + recommendations per
> pipeline stage. Does **not** rewrite code. Adoption decisions remain
> operator-gated.
> **Companion:** `plans/wiki_layer_integrated.md` is the canonical
> wiki/cross-disciplinary design; this document covers the *rest* of
> the stack and the wiki layer's alternatives.

## Executive summary

After surveying the April-2026 landscape, the **current Medii architecture
is mostly well-chosen** but has four places where credible alternatives
are worth weighing:

| Stage | Current | Strongest alternative | Verdict |
|---|---|---|---|
| **Extraction** | Docling + PyMuPDF4LLM | **MinerU** for image-heavy PDFs | Keep Docling as primary; add MinerU for image-heavy fallback |
| **Knowledge layer** | Karpathy LLM Wiki (in progress) | Hybrid Wiki + RAG (current vector index retained) | **Already aligned** — Medii's design keeps both |
| **Cheap cascade tier** | DeepSeek V4-Flash ($0.14/$0.28) | Qwen 3.5 9B ($0.05) free for low-volume; o4-mini for JSON reliability | Keep V4-Flash for bulk; **switch wiki editor to o4-mini** (bake-off-validated) |
| **Embeddings** | BGE-M3 (FP16, 230 chunks/sec) | **Qwen3-Embedding-8B** (70.58 MTEB, multilingual leader) | Stay on BGE-M3 unless retrieval recall measurably plateaus |
| **Local LLM** | MedGemma 4b / Qwen 2.5 7b (offline only) | **MedGemma 4b for vision triage** + nothing else | Demote text local to drill-only; keep MedGemma for vision |
| **Vector store** | ChromaDB | pgvector (if you ever need joins to metadata) | Keep Chroma — under 1M chunks |
| **Cascade orchestration** | Custom Python in `src/oversight/router.py` | **LiteLLM as proxy + custom logic on top** | Stay custom; LiteLLM only if you grow past 4 providers |
| **Medical NER** | None | **MedCAT** (UMLS / SNOMED-CT linking) | Add for wiki audit + variant under-citation detection |
| **Structured output** | Hand-rolled JSON parsing | **Instructor** (3M monthly downloads, Pydantic-native) | Adopt incrementally — start with wiki ingest editor |
| **Chunking** | Naive paragraph split (in `02_embed.py`) | **Adaptive + late chunking** (87% vs 13% on medical) | High-impact change; recommended after wiki layer lands |

The single most important decision is **Karpathy LLM Wiki vs alternatives** —
that's the architectural fork the project is at right now. The TL;DR
from the research:

> *"Karpathy's LLM Wiki is more like building an encyclopedia for a person.
> Graphify [GraphRAG] is more like building a map for an agent."* —
> [AI-Chain analysis](https://ai-chain.tw/en/blog/graphify-vs-karpathy-llm-wiki-knowledge-base-guide/)

Medii's case-drafter agent is a *machine* reading wiki pages to author
patient cases for *humans* (medical students). That makes the choice
non-obvious. Both designs satisfy the project goals; the current
Wiki + retained-vector-index hybrid is a reasonable compromise.

---

## Goals this document optimises for

Anchored to the operator's stated objectives (verbatim from prior
sessions):

1. **Clinical accuracy is invariant #1.** Every architectural choice is
   weighed against "could this hallucinate in a way that misleads a
   medical student?"
2. **Cross-disciplinary integration.** Clinical + history + biomechanics
   + art + nature, woven into how concepts are taught — not dangling
   asides.
3. **Subtype awareness.** Posterior STEMI must surface *its* ECG, not
   the anterior STEMI ECG. Drafter must be requestable by variant.
4. **Cost discipline.** £150 ($190) hard cap; cheap cascade tier handles
   ≥95% of calls; frontier sees flagged + sampled-PASS only.
5. **RTX 3070 (8GB VRAM) + iGPU display.** Local stack has to fit in
   that budget without contending for compute.
6. **Markdown-first, git-versioned.** No graph DB, no proprietary store,
   plain `.md` files in git. Karpathy's slogan: "the wiki is the codebase."
7. **Mirror-friendly.** Notes are readable from a tablet via the
   sibling `Medii_Markdown_Mirror` repo.

A choice that wins on cost but loses on (1) is rejected. A choice that
wins on (1) but multiplies cost 10× is rejected. The cascade pattern
(cheap executor + sampled deep verifier) is how Medii reconciles 1+4.

---

## Stage 0 — Knowledge layer (the architectural fork)

### What's at stake

This is the highest-leverage choice in the whole pipeline. It dictates
how case-drafter cost, accuracy, freshness, and cross-disciplinary
integration *all* behave.

### The four credible designs

#### Design A — Plain RAG (the original Medii through 2026-04-24)

```
sources → chunks → embeddings → vector index → retrieve top-K → drafter
```

- **Pro:** zero compounding cost — embed once, retrieve forever.
- **Pro:** vector retrieval handles "find verbatim quote" lookups well.
- **Con:** knowledge does NOT compound across sources. The cardiology
  corpus doesn't surface its own internal consistency.
- **Con:** drafter re-discovers concept structure on every case run.
- **Con:** subtype handling is ad-hoc — drafter prompted with chunks,
  hopes they cover the requested variant.

**Verdict:** what Medii outgrew on 2026-04-25. Still useful as the
*verbatim quote* fallback layer.

#### Design B — Karpathy LLM Wiki (current Medii direction)

```
sources → chunks → vector index (retained) → drafter
              ↓
       wiki ingest pipeline → corpus/wiki/{sub}/<concept>.md → drafter
                                                                  ↑
                                                       reads wiki page
                                                       FIRST, falls back
                                                       to vector index
                                                       for verbatim quotes
```

Concept-organised compounded knowledge per
[Karpathy's April-2026 gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f),
extended with cross-disciplinary sub-corpora and subtype variants per
`plans/wiki_layer_integrated.md`.

- **Pro:** knowledge compounds — the cardiology *page* has the consensus
  of all sources cited side by side.
- **Pro:** subtype variants are first-class structure
  (`stemi.variants.posterior` with own ECG references).
- **Pro:** cross-disciplinary integration is natural —
  `clinical/stemi.md` cross-links to `art/davinci_heart_studies.md` via
  YAML frontmatter, drafter pulls into `optionalExtensions[]`.
- **Pro:** plain markdown in git, parseable by any editor, mirror-friendly.
- **Pro:** *human-readable* knowledge base — a medical educator can
  open `corpus/wiki/clinical/stemi.md` and review what the model believes.
- **Con:** ingest cost — every source costs ~5-8 cascade calls
  (~$0.05-$0.10 per BMJ guideline).
- **Con:** drift risk — pages can decay from sources over time without
  per-claim provenance enforcement (mitigated by `wiki/audit.py`).
- **Con:** scale ceiling — Karpathy explicitly says ~100K tokens per
  wiki, after which you need vector search again. Medii's UK MLA
  curriculum is ~250 conditions; well within the ceiling.

**Verdict:** the right choice for Medii's scope. Design B is what's in
flight on `claude/wiki-layer`.

**Important caveat from research:**
[Atlan](https://atlan.com/know/llm-wiki-vs-rag-knowledge-base/)
notes Karpathy's pattern is *"explicitly scoped to individual
researchers."* Medii is closer to that scale (one operator, ~250
concepts, ~30-50 sources) than to enterprise scale (millions of docs).
The pattern fits.

#### Design C — GraphRAG / Graphify (knowledge graph)

```
sources → chunks → entities + relationships → graph DB (Neo4j / etc)
                                                  ↓
                                          drafter queries graph
```

- **Pro:** [Graphify benchmark](https://www.analyticsvidhya.com/blog/2026/04/graphify-guide/)
  shows **71.5× fewer tokens per query** vs naive document loading.
- **Pro:** queries like "everything connecting STEMI to cardiogenic
  shock to inotropes" are native — the graph traversal IS the answer.
- **Con:** requires a graph DB (Neo4j / Memgraph / Kuzu) — adds infra,
  breaks the "wiki is the codebase" invariant.
- **Con:** edges hallucinated by the LLM can quietly compound errors.
  Medical edges (drug-drug interactions, contraindications) are
  *exactly* where hallucination is dangerous.
- **Con:** cross-disciplinary integration is awkward — Da Vinci's heart
  drawings as a graph node feels forced; as a wiki cross-link it's
  natural.
- **Con:** the graph is **for an agent**, not a human. Medii's operator
  needs to be able to spot-check.

**Verdict:** rejected for Medii. The token-saving pitch is strong, but
the failure modes (silent edge hallucination + opacity to the operator)
are the wrong trade-offs for clinical content. Reconsider only if Medii
ever pivots from "case authoring" to "agent that answers clinical
questions in real time."

#### Design D — Hybrid Wiki + GraphRAG (what some enterprises do)

Wiki for human-readable concept summaries, graph for entity
relationships, vector index for verbatim quotes. Three layers.

- **Pro:** strictly more powerful than B alone.
- **Con:** 3× the maintenance burden. 3 sync states to manage.
- **Con:** the graph layer's failure modes (silent edge hallucination)
  apply here too.
- **Con:** overkill for ~250 concepts.

**Verdict:** revisit if Medii grows past ~10K concepts (would need
to expand outside UK MLA into international curricula + paramedicine
+ pharmacy). Not for the foreseeable future.

### Recommendation

**Stay with Design B (Karpathy LLM Wiki + retained vector index).**
It's what's in flight, it matches operator goals, and the failure modes
are operator-debuggable. The research validates the choice for projects
in Medii's size range.

The next session must finish the STEMI prototype rerun + decision gate
before sinking more effort into the wiki direction.

---

## Stage 1 — PDF→Markdown extraction

### Current Medii choice

- `src/01_ingest.py` — **PyMuPDF4LLM** (fast, text-only, flattens tables).
- `src/01b_docling_extract.py` — **Docling + Granite-Docling-258M**
  on RTX 3070 (preserves tables/math/layout).

Operator selects per-source: PyMuPDF4LLM for text-rich BMJ guidelines;
Docling for image- and table-heavy textbooks.

### Alternatives surveyed

| Tool | Strength | Weakness | Medical fit |
|---|---|---|---|
| **Docling** (IBM) | DocLayNet + TableFormer; preserves complex tables; integrates Tesseract/EasyOCR/RapidOCR | Slower than PyMuPDF4LLM; needs GPU for best results | **Strong** — preserves diagnostic-criteria tables |
| **Marker** | Fast (GPU/CPU/MPS); Surya OCR; structured markdown; multilingual | Closed `datalab.to` ecosystem creeping in; less battle-tested at table preservation | Good general-purpose; weaker on complex medical tables |
| **MinerU** (OpenDataLab) | "Best on rotated tables and complex academic papers" — excellent at preserving document structure; embeds tables as HTML | Slower; heavier model stack | **Strong on image-heavy PDFs** (textbooks with diagrams) |
| **PyMuPDF4LLM** | Fastest; pure-python; no GPU; battle-tested | Flattens tables; loses layout for image-heavy pages | Fast text path only |
| **Mistral OCR API** | Cloud API; high accuracy; handles handwriting | Costs per page; sends medical PDFs to cloud (data residency concern) | Avoid for clinical content unless self-hosted |
| **Nougat** (Meta) | Specialised for academic papers + math | Slow; LaTeX-focused; superseded by Docling | Niche |
| **Unstructured.io** | Big ecosystem; many file formats | Less specialised on tables; cloud option costs | Generic |
| **MarkItDown** (Microsoft) | Lightweight; many formats | Weak on complex layouts | Generic |

### Trade-offs that matter

- **Table preservation is non-negotiable for clinical content.** STEMI
  diagnostic criteria live in tables; flattening them to prose loses
  the "≥2 mm in V2-V3 for men >40, ≥1.5 mm for women" structure.
- **GPU contention.** Docling on `cuda:0` blocks BGE-M3 embedding work
  per ARCHITECTURE.md's sequential rule. Acceptable; just sequence the
  pipeline.
- **PWA-runtime extraction is not in scope.** Extraction runs once per
  source, output is committed to `corpus/extracted_md/`. Speed matters
  less than fidelity.

### Recommendation

**Keep Docling as the primary preservation-quality extractor; keep
PyMuPDF4LLM as the speed path; add MinerU for image-heavy textbook
fallback** (Robbins Basic Pathology, the ECG workbook).

Per [ThemenOnLab 2026 review](https://themenonlab.blog/blog/best-open-source-pdf-to-markdown-tools-2026):
*"For complex academic papers with CJK text, MinerU is recommended.
For enterprise RAG, Docling is preferable."* Medii's textbooks are the
"complex academic papers" case.

Effort: ~2 hours to add a `src/01c_mineru_extract.py` that routes by
heuristic (image-density threshold).

---

## Stage 2 — Chunking

### Current Medii choice

`src/02_embed.py` uses a simple paragraph-based split. Window size
isn't configurable; no semantic awareness.

### Alternatives surveyed

| Strategy | Description | Medical-corpus result |
|---|---|---|
| **Fixed-size** (current Medii) | Split every N tokens with M overlap | 13% accuracy on medical CDS benchmark |
| **Semantic chunking** | Embed sentences, cluster by similarity | 54% — but produces tiny fragments (~43 tokens) |
| **Adaptive chunking** | Respect document structure (headings, sections) | **87%** on medical CDS (p=0.001 vs fixed) |
| **Proposition chunking** | Extract atomic facts as standalone units | High retrieval precision; low coverage |
| **Late chunking** ([JinaAI](https://jina.ai/news/late-chunking-in-long-context-embedding-models/)) | Embed long context first, chunk after | Preserves cross-reference context |
| **Contextual retrieval** ([Anthropic](https://www.anthropic.com/news/contextual-retrieval)) | Add doc-level summary to each chunk before embedding | +35% recall@K vs naive |
| **Graph-aware late chunking** | Late chunking + entity graph | Biomedical-specific; SOTA on bio benchmarks |

### Trade-offs

- **Adaptive chunking is the obvious win** for medical content. Headings
  in BMJ guidelines and textbooks are *meaningful* — splitting by
  heading boundaries respects clinical structure (Diagnosis / Management
  / Complications).
- **Contextual retrieval is cheap and high-impact.** Adds ~$0.001 per
  chunk (one short LLM call to summarise heading-path context); recall
  improves measurably.
- **Late chunking and proposition chunking are stretch goals** —
  worthwhile only after adaptive + contextual are in place.

### Recommendation

**Add adaptive chunking + contextual retrieval to `src/02_embed.py`**
as a follow-up to the wiki layer. This is a high-impact change:

- Adaptive chunking: respect `## Heading` boundaries during split.
  ~1 day of work.
- Contextual retrieval: prepend `[<doc title> > <heading path>]` to each
  chunk before embedding. Cheap cascade call per chunk. ~1 day.

Combined effect: per the [PMC 2026 study](https://pmc.ncbi.nlm.nih.gov/articles/PMC12649634/),
medical-corpus retrieval accuracy goes from baseline to >85%.

---

## Stage 3 — Embeddings

### Current Medii choice

**BGE-M3** in FP16 on the RTX 3070 (cuda:0). 230 chunks/sec.
~1.2 GB load + ~2.8 GB peak.

### Alternatives surveyed (April 2026 MTEB leaderboard)

| Model | Avg MTEB | Notes | Local-on-3070? |
|---|---|---|---|
| **NVIDIA Llama-Embed-Nemotron-8B** | Top of multilingual MTEB | Requires 8B model; tight fit | Marginal in FP16 |
| **NV-Embed-v2** | 72.31 (English) | **CC-BY-NC-4.0 — no commercial use** | Yes but licence kills it |
| **Qwen3-Embedding-8B** | 70.58 (multilingual) | Flexible dims 32→4096; 100+ languages | Yes; tight |
| **Gemini Embedding 001** | 68.32 (English) | Cloud-only API | Cloud-only |
| **Gemini Embedding 2 Preview** | (preview) | Multimodal: text+image+video+audio+PDF | Cloud-only; potentially game-changing for image triage |
| **BGE-M3** (current) | ~67-68 multilingual | 8192 token window; 100+ langs; multi-granularity | **Yes — currently runs at 230 chunks/sec FP16** |
| **Stella-en-1.5B-v5** | 71.19 (English) | English-only; smaller | Yes, easy fit |
| **mxbai-embed-large-v1** | 67.x | German team; production-quality | Yes |
| **OpenAI text-embedding-3-large** | 64.6 | Cloud API; expensive | Cloud-only |

### Trade-offs

- **BGE-M3 is still very competitive** in 2026, particularly for
  multilingual + multi-granularity. The newer Qwen3-Embedding-8B beats
  it by ~3 points on MTEB but costs 2-3× the VRAM.
- **The interesting jump is multimodal embedding** — Gemini Embedding 2
  preview can embed images natively. Could replace the entire
  image-triage stage with vector similarity ("how close is this
  image's embedding to the embedding of the surrounding clinical
  text?"). But cloud-only and preview-status; not yet production.
- **NV-Embed-v2's CC-BY-NC-4.0 licence is disqualifying** — Medii is
  building toward a deployable PWA. Cannot ship NC content.

### Recommendation

**Stay on BGE-M3.** It's not the leaderboard champion but it's the
right balance of quality, multilingual coverage, VRAM fit, and licence
freedom. Re-evaluate when:

1. The wiki layer is in steady state and `qrels_baseline.json` shows
   recall@K plateauing below 0.8 (the embed gate threshold).
2. OR Gemini Embedding 2 graduates from preview AND the image-triage
   stage becomes a measurable bottleneck.

Effort to swap: ~4 hours (download new model, update `02_embed.py`'s
`MEDII_EMBED_MODEL` env var, re-embed, re-baseline qrels).

---

## Stage 4 — Vector store

### Current Medii choice

**ChromaDB** persisted at `db/vector_index/`.

### Alternatives surveyed

| Store | Scale (vectors) | Latency p50 | Cost @ 50M | Best for |
|---|---|---|---|---|
| **ChromaDB** (current) | <1M (recommended) | ~10ms | <$30/mo single VPS | Default for new projects |
| **Qdrant** | Billions (distributed) | **4ms** | ~$$ | Lowest latency; complex filtering |
| **LanceDB** | 10M+ | ~8ms | Disk-efficient via Lance format | Larger-than-RAM datasets |
| **pgvector / pgvectorscale** | 10-50M single node | ~6ms | **75% cheaper than Pinecone** at 50M | Existing Postgres shops |
| **Milvus** | Billions | ~6ms (GPU) | High infra | Enterprise scale |
| **Pinecone** | Billions (managed) | ~8ms | Premium | Hands-off managed |
| **Weaviate** | Billions | ~8ms | Mid | Hybrid search built in |
| **Vespa** | Billions | ~5ms | Complex | Search engine + vectors |

### Trade-offs

- **Medii's corpus is small.** 22 source files → ~30K chunks today,
  scaling to maybe ~500K chunks if all of UK MLA is ingested. That's
  squarely in ChromaDB's sweet spot.
- **The pgvectorscale story is interesting** if Medii ever grows the
  PWA into a Postgres-backed application — could collapse vector +
  metadata + cases into one DB. But it's premature optimisation today.
- **Qdrant's 4ms p50 doesn't matter** at Medii's call volume.

### Recommendation

**Keep ChromaDB.** Re-evaluate only if:

1. Chunk count crosses 1M (add Qdrant or LanceDB), OR
2. Medii adds a Postgres-backed PWA backend (consider migrating to
   pgvector for join-with-metadata queries).

---

## Stage 5 — Cascade orchestration (cloud LLM tiers)

### Current Medii choice

| Tier | Role | Model | Price ($/MTok) |
|---|---|---|---|
| 1 | cheap_judge | `deepseek/deepseek-v4-flash` | 0.14 / 0.28 |
| 2 | bulk_auditor | `z-ai/glm-5.1` | 1.05 / 3.50 |
| 3 | deep_critic | `deepseek/deepseek-v4-pro` | 1.74 / 3.48 |

Routed through OpenRouter (single API key, three independent providers).
Anthropic + OpenAI direct as secondary fallback.

### April-2026 cloud LLM landscape

#### Cheap tier candidates

| Model | $/MTok in/out | Notes |
|---|---|---|
| **Qwen 3.5 9B** | $0.05 / ~$0.20 | Cheapest reliable model on OpenRouter |
| **DeepSeek V4-Flash** (current) | $0.14 / $0.28 | Bake-off-validated for short prompts; **fails on long nested JSON (the STEMI prototype blocker)** |
| **Gemini 2.5 Flash** | $0.15 / $0.60 | Google quality; output 2× more expensive |
| **DeepSeek V3.2** | $0.27 / $1.10 | Older but reliable |
| **Kimi K2.6** | $0.60 / $2.80 | Multimodal; agentic; "Quality Index 53.9 leads open-source" |
| **GPT-4.1 mini** | varies | OpenAI cheap tier; reliable JSON |
| **Claude Haiku 4.5** | $1.00 / $5.00 | Premium-cheap; vision-capable; reliable |

#### Cross-check tier candidates

| Model | $/MTok in/out | Notes |
|---|---|---|
| **GPT-4.1 mini** | varies | |
| **o4-mini** | $1.10 / $4.40 | **Bake-off champion for JSON reliability — never returned None** |
| **GLM-5.1** (current) | $1.05 / $3.50 | |
| **DeepSeek V3** | $0.27 / $1.10 | |

#### Deep tier candidates

| Model | $/MTok in/out | Notes |
|---|---|---|
| **DeepSeek V4-Pro** (current) | $1.74 / $3.48 | 1.6T MoE; 1M context; **subject to Together rate limits** |
| **Claude Opus 4.7** | $15 / $75 | Premium; multimodal; reliable; expensive |
| **GPT-5** | varies | |
| **Gemini 2.5 Pro** | varies | |
| **DeepSeek R1** | varies | Strong reasoning |

#### Free tiers

[Per costgoat.com](https://costgoat.com/pricing/openrouter): DeepSeek
R1, Llama 3.3 70B, and Gemma 3 are available at **zero cost** on
OpenRouter; Qwen 3.6 Plus is fully free during preview with a
**1M context window**. Useful for low-volume drilling but rate-limited.

### Trade-offs

- **The bake-off finding is decisive for the wiki editor failure:
  o4-mini was the most reliable JSON producer.** V4-Flash's $0.14 cost
  advantage doesn't matter if it returns None on the long editor
  prompt.
- **Multi-vendor cascade is the right design.** Three independent
  providers means a single-vendor outage or silent regression cannot
  pass through unnoticed. OpenRouter as the single key drastically
  reduces operational burden.
- **DeepSeek V4-Pro's Together rate limits** are a real risk — already
  hit in Stage 4 case drafter. Mitigated by the 3-tier planner fallback
  chain (`deep` → `bulk_alt` → `cheap`); should keep that pattern
  everywhere.

### Recommendations

1. **Switch the wiki editor tier from `cheap` → `bulk_alt` (o4-mini)
   for `wiki_ingest` only.** Config change in `config.yaml`. Costs go
   up by ~6× per call, but call volume is low (one editor call per
   page per ingest run; ~30-50 pages total over the project lifetime).
   Total cost impact: maybe $0.30-$0.50 across the entire wiki ingest
   lifecycle. **Worth it for reliability.**

2. **Add Qwen 3.5 9B as a "drill mode" cheap_judge alternative** —
   $0.05/MTok is a 3× cost reduction vs V4-Flash if it works for short
   QC prompts. Drill against seed_samples first.

3. **Keep V4-Pro as deep_critic** despite the rate limits — the 1M
   context is genuinely useful for the wiki editor's longest prompts;
   nothing else in the lineup matches it at that price point.

4. **Don't bother with paid OpenAI o4 / Claude Opus** unless V4-Pro's
   rate limits become a daily problem. The cost premium (10-30×) isn't
   justified for Medii's verification workload.

#### Medical-specialised cloud models

| Model | MedQA score | Where to use |
|---|---|---|
| **MedGemma 1.5** (Jan 2026) | **~91%** | SOTA open-weight medical model |
| **MedGemma 27B** (text) | 87.7% | Within 3 pts of DeepSeek R1 at 1/10 cost |
| **MedGemma 4B** (vision) | — | **81% of chest X-ray reports judged equivalent to radiologist** |
| **Med-PaLM 2** | 86.5% | Closed; Google Cloud only |
| **Meditron 70B** | ~70% | Open; EPFL/Yale; PubMed-tuned |

**Important:** MedGemma 27B at 87.7% MedQA *beats* DeepSeek V4-Pro on
medical-specific reasoning, despite costing ~1/10 as much per token.
**This is a genuine architecture-question.** Should the deep_critic for
*medical* cases be MedGemma 27B (specialised, cheap) instead of V4-Pro
(general, expensive)?

**Counter-argument:** The cascade's deep tier audits *clinical text
quality* (does the page say what the source says?), not *medical
reasoning* (what should the diagnosis be?). For text-quality audits,
the general-purpose V4-Pro is appropriate. For *case authoring*
(Stage 4 drafter), specifically the verifier step that checks "does
this case make clinical sense", MedGemma 27B becomes a strong candidate.

**Recommendation:** Add MedGemma 27B (via `google/medgemma-27b` on
OpenRouter when it lands there, or self-hosted on a beefier box if
not) as the **case-drafter verifier** in `stages.case_draft.verifier`.
Keep V4-Pro for wiki-page-quality audits. ~1 day of work after wiki
prototype passes its decision gate.

---

## Stage 6 — Local LLM role on the RTX 3070

### Current Medii choice

`.venv-ml/` (Py 3.12) on the 3070 runs:
- **BGE-M3 FP16** (embeddings) — primary local workload.
- **MedGemma 4B** (vision triage) — ~3 GB VRAM; fast.
- **Qwen 2.5 7B** (text judge, drill mode) — ~5 GB VRAM Q4_K_M; slow.
- **Granite-Docling-258M** (PDF extraction) — ~0.6 GB VRAM.

`MEDII_OFFLINE=1` forces local-only mode; cascade returns to Ollama
fallback.

### April-2026 8GB-VRAM landscape

Per [LocalLLM.in benchmarks](https://localllm.in/blog/best-local-llms-8gb-vram-2025):

| Model | Quant | VRAM | tokens/sec | Verdict |
|---|---|---|---|---|
| **Qwen 3.5 9B** | Q4_K_M | 6.96 GB peak (32K ctx) | 54-58 | **Best general-purpose 8GB winner** |
| **Phi-4 14B** | Q4 | spills to RAM at 32K | 1.7 (effectively unusable) | Skip |
| **Phi-4 Mini 3.8B** | Q4 | 2.3 GB | very fast | Coding-focused; weaker on medicine |
| **Gemma 3 9B** | Q4_K_M | ~5.5 GB | ~50 | Strong general; less specialised |
| **Llama 3.2 3B** | Q4 | ~2 GB | very fast | Too small for judgement |
| **MedGemma 4B** (current) | Q4 | ~2.5 GB | ~30 | **Vision-capable; medical-tuned; right fit** |
| **MedGemma 27B** | Q4 | ~17 GB | OOM | Cannot run on 8GB |
| **Qwen 2.5 7B Instruct** (current) | Q4_K_M | ~5 GB | ~25 | Outclassed by Qwen 3.5 9B |

### Trade-offs

- **The 3070 cannot run a judge-grade text LLM.** Even Qwen 3.5 9B
  at 54-58 tok/s is too slow for cascade volumes (V4-Flash via
  OpenRouter does ~200+ tok/s and costs $0.001 per call). Local text
  inference is **drill mode only** and accepted as such.
- **MedGemma 4B vision is the clear local winner.** Vision model API
  calls are expensive ($0.001-$0.005 per image); doing ~1000 image
  triages locally saves real money and avoids data-residency concerns.
- **Embeddings are the right primary local workload.** 230 chunks/sec
  on FP16, no API cost, no network latency.

### Recommendations

1. **Demote text local judges to drill-only.** Already done in the
   cloud-first pivot; just confirm `MEDII_OFFLINE=1` is the only path
   that triggers them.
2. **Keep MedGemma 4B for vision triage** as the primary path
   (`stages.triage.judge: vision_cheap` already routes here when
   OpenRouter is unavailable for vision).
3. **Upgrade the local text fallback from Qwen 2.5 7B → Qwen 3.5 9B
   Q4_K_M** when next touching `local_judge.py`. ~30 min of work; no
   architectural risk.
4. **Do NOT attempt to run MedGemma 27B locally.** The 8GB ceiling is
   real; even Q4 is 17 GB. Use MedGemma 27B *via cloud* (OpenRouter or
   Together) for case-drafter verification.

---

## Stage 7 — Cascade orchestration framework

### Current Medii choice

Custom Python in `src/oversight/router.py` (`judge_text` / `judge_vision`
dispatch) calling `src/oversight/frontier.py` (provider clients).
~150 lines of router code; transparent and debuggable.

### Alternatives surveyed

| Framework | Role | Trade-off |
|---|---|---|
| **LiteLLM** | Proxy server normalising 100+ providers; fallbacks; budget tracking | Heavy (Postgres for spend tracking; UI); recent supply-chain compromise (Trend Micro Mar 2026); useful only if Medii grows past 4-5 providers |
| **DSPy** | Programmatic prompt optimisation + module composition | Strong for *optimising* prompts; less useful for routing logic itself; non-trivial learning curve |
| **Instructor** | Structured-output extraction with Pydantic validation | **High value** — Medii already uses Pydantic schemas; Instructor would replace ~50 lines of hand-rolled JSON parsing |
| **Outlines** | Token-masking FSM for guaranteed-valid JSON output | Strong for self-hosted models; less useful for cloud APIs that already support `response_format=json_object` |
| **PydanticAI** | Typed agent runtime (replayable datasets, evals) | Newer; aimed at "agent" workflows more than cascade |
| **RouteLLM** | Cost-optimal model routing per prompt complexity | Academic — interesting for adaptive routing but Medii's tier rules are intentionally explicit |
| **Mirascope** | Pythonic LLM call wrapping | Smaller community than Instructor |

### Trade-offs

- **Custom Python is winning** at Medii's scale. The router is 150 lines,
  clear, debuggable. Adding LiteLLM would replace that with another
  150 lines of config + a Postgres dependency.
- **Instructor is the credible adoption candidate.** It's Pydantic-native
  (Medii already uses Pydantic in `src/oversight/schemas.py` and
  `src/wiki/schemas.py`), 3M monthly downloads (battle-tested), and
  would clean up the JSON-parse retry logic that just had to be
  manually hardened in `frontier.py`. The new `_balanced_json_substring`
  function does what Instructor does for free.
- **DSPy is overkill** unless Medii starts doing prompt-optimization
  experiments (compiling prompts via gradient-style updates against a
  training set).

### Recommendations

1. **Adopt Instructor incrementally** — start with the wiki ingest
   editor call (the one that was failing) since that's where structured
   output reliability matters most. ~2-4 hours of work to wrap one
   `judge_text` call. Measure JSON parse failure rate before/after.
2. **Skip LiteLLM** unless Medii adopts >5 providers (today: 3 + 2
   fallbacks = 5; right at the threshold but not over it).
3. **Skip DSPy** unless prompt-optimisation experiments become a
   workstream.
4. **Skip Outlines** unless local-LLM JSON reliability becomes a
   problem (currently solved by cloud `response_format=json_object`).

---

## Stage 8 — Medical-specific tooling

### Current Medii choice

**None.** Medii treats medical text as generic text. No NER, no
ontology linking, no clinical-concept normalisation.

### Alternatives surveyed

| Tool | Role | Maintainability | Medical fit |
|---|---|---|---|
| **MedCAT** (CogStack) | Self-supervised concept extraction; UMLS / SNOMED-CT linking | Active; UK-NHS-derived; current models include SnomedCT UK Clinical edition June 2025 | **Strongest** — UK-specific |
| **scispaCy** (Allen AI) | spaCy models trained on biomedical corpora | Active; integrates with MedCAT | Strong general |
| **MetaMap / MetaMapLite** | Classic UMLS NER (NLM) | Less active; Java-only | Legacy |
| **cTAKES** (Apache) | clinical Text Analysis and Knowledge Extraction | Active but heavy; Java | Enterprise-scale |
| **Stanza Biomedical** (Stanford) | Biomedical NLP toolkit | Active | Strong for research; less UK-focused |

### Use cases for Medii

1. **Wiki audit enhancement.** When `wiki/audit.py` walks a page,
   currently it only checks `[N]` footnote presence. With MedCAT, it
   could also check that **every drug mentioned in `## Management (UK)`
   is a real BNF entity**, that every diagnosis is a SNOMED-CT
   concept, etc. Catches hallucinated drug names + nonexistent
   conditions.
2. **Variant under-citation detection.** The plan defines
   `VARIANT_UNDERCITED` finding for variants with <2 sources. With
   MedCAT, you can check that the variant's *clinical entities*
   (diagnoses, drugs, signs) are each cited too — much stronger
   provenance audit.
3. **Cross-link proposal.** Auto-suggest cross-disciplinary links by
   finding shared concepts: "this `clinical/stemi.md` mentions LAD
   coronary artery; this `art/davinci_heart_studies.md` also mentions
   LAD coronary artery → propose cross-link."
4. **MLA syllabus gap reporting.** Auto-tag concepts to MLA codes; the
   master plan's Stage 7 curriculum auditor needs this anyway.

### Trade-offs

- **Adds a dependency** but it's a Python package (heavy on first model
  load, then fine). No new services or DBs.
- **Models are large** — full SnomedCT UK Clinical model is ~2GB. Loads
  once, cached. Acceptable.
- **Doesn't run on the 3070** — CPU-only. Doesn't contend with BGE-M3.

### Recommendation

**Add MedCAT in a follow-up after wiki layer lands.** Use case #1
(wiki audit drug-name + diagnosis verification) is the highest-impact
single addition Medii could make for clinical safety. ~1-2 days of
work to wire it into `src/wiki/audit.py` as an additional finding code
(`UNRECOGNISED_DRUG`, `UNRECOGNISED_DIAGNOSIS`).

---

## Stage 9 — Image triage + cross-disciplinary content sources

### Current Medii choice

`src/04_image_triage.py` uses **MedGemma vision via Ollama** locally;
Opus 4.7 vision via Anthropic for grey-zone tie-break.

For cross-disciplinary content sources (Da Vinci, Corrigan, etc.),
**no source pipeline exists** — pages are hand-drafted by the operator
per the integrated plan.

### Public-domain / openly-licensed image sources

| Source | Licence | Strength | Coverage |
|---|---|---|---|
| **Wellcome Collection** | **CC-BY 4.0** | 100K+ images; medicine + history | Strongest single source for Medii's needs |
| **Met Museum (Open Access)** | **CC0** (public domain) | 492K+ images | Da Vinci sketches, anatomical art |
| **Public Domain Review** | curated PD | High-quality picks | Editorial quality |
| **Royal Collection Trust** | mixed; some PD | Da Vinci's heart anatomy specifically | Need per-image check |
| **Wikimedia Commons** | mixed; CC family | Catch-all | Need per-image check |
| **Smithsonian Open Access** | CC0 | Large; varied | Some medical |
| **NIH Open-i** | varied | Biomedical literature images | **Clinical relevance** |
| **Library of Congress** | mostly PD | Historical medical photos | Strong on US medical history |

### Recommendation

Build a `corpus/image_assets/` ingestion CLI alongside the wiki
ingest:

```bash
python tools/wiki_image_ingest.py --source wellcome --query "heart anatomy" \
    --target corpus/wiki/art/davinci_heart_studies.md
```

Each ingested image gets a sidecar JSON manifest with:
- `source` (wellcome / met / loc / etc)
- `license` (CC-BY 4.0 / CC0 / etc)
- `url` (canonical link)
- `attribution` (required even for CC-BY)
- `accession_id`

The pre-flight check that the integrated plan's pitfall #11 calls for
("each `image_assets:` entry must reference a manifest with a `license`
field") trivially reads from these sidecars.

**Effort:** ~1 day for the Wellcome + Met API integrations. Defer
Royal Collection (no clean API) until needed.

---

## Architecture-level alternatives (whole-pipeline shapes)

### Variant 1 — Current Medii (pipeline of CLIs + cascade harness)

```
src/01_ingest.py  →  src/02_embed.py  →  src/03_qc_audit.py
                                              ↓
src/04_image_triage.py  →  src/wiki/ingest.py  →  src/05_case_drafter.py
                                              ↑
                            src/oversight/router.py (cloud cascade)
```

- **Pro:** each stage is a standalone CLI; can be re-run independently;
  pause/resume per stage.
- **Pro:** debuggable — operator can inspect output at each stage
  boundary.
- **Pro:** no orchestration framework dependency.
- **Con:** state is implicit (file paths in `corpus/`).
- **Con:** no built-in DAG visualisation.

### Variant 2 — DAG orchestration (Prefect / Dagster / Kedro / Argo)

- **Pro:** explicit DAG; visualisation; retry + observability.
- **Con:** adds a service to run; learning curve; over-engineering for
  a one-operator project.

**Verdict:** stay with Variant 1. Re-evaluate if Medii ever becomes a
multi-operator team or a hosted service.

### Variant 3 — Notebook-driven (Marimo / Jupyter)

- **Pro:** interactive, visual, easy to spot-check.
- **Con:** state lives in the kernel; reproducibility harder.

**Verdict:** appropriate for *experimentation* (e.g. wiki page diffing,
qrels exploration). Not appropriate for the production pipeline.

### Variant 4 — Background-agent assisted (Claude Code itself, Cursor agents)

- **Pro:** Medii is already heavily AI-pipeline-assisted; doubling down
  on agent-driven evolution is natural.
- **Con:** drift risk — agents can re-introduce stale patterns. The
  current AI-rule-file harmonisation pass exists exactly because of
  this drift.

**Verdict:** keep using AI agents for code edits, but the *pipeline
runtime itself* should be deterministic Python + cascade calls. Don't
make agents part of the runtime path.

### Variant 5 — Online API service (FastAPI + queue)

- **Pro:** the PWA can call a live API instead of static markdown.
- **Con:** completely changes the Medii's "wiki is the codebase"
  identity; turns it into a backend service.

**Verdict:** out of scope. The PWA reads checked-in markdown +
case JSON files; that's the design.

---

## Recommended adoption roadmap

In strict priority order, only after current STEMI prototype crosses
its decision gate:

| # | Change | Cost | Impact | Effort |
|---|---|---|---|---|
| 1 | **Switch wiki editor tier to o4-mini** for `wiki_ingest` only | +$0.30-0.50 lifetime | Unblocks STEMI prototype | 5 minutes (config-only) |
| 2 | **Adopt Instructor** for the wiki ingest editor call | $0 | JSON reliability + ~50 lines of hand-rolled code removed | 2-4 hours |
| 3 | **Add MinerU** as image-heavy PDF fallback alongside Docling | $0 | Better tables in textbooks | 2 hours |
| 4 | **Add adaptive chunking** to `02_embed.py` | $0 | Major retrieval-quality improvement | ~1 day |
| 5 | **Add contextual retrieval** to `02_embed.py` | ~$0.01 per chunk one-time | +35% recall@K | ~1 day |
| 6 | **Add MedCAT** to `wiki/audit.py` | $0 | Catches hallucinated drugs / diagnoses | ~1-2 days |
| 7 | **Switch case-drafter verifier to MedGemma 27B** (cloud) | -90% verifier cost | Stronger medical reasoning | ~4 hours |
| 8 | **Build `tools/wiki_image_ingest.py`** (Wellcome + Met) | $0 | Cross-disciplinary content with proper licensing | ~1 day |
| 9 | **Upgrade local text fallback to Qwen 3.5 9B** | $0 | Better drill-mode quality | 30 min |
| 10 | **Re-evaluate embeddings** (BGE-M3 → Qwen3-Embedding-8B) only after recall@K plateau | $0 | Modest MTEB gain; uncertain real-world impact | ~4 hours when triggered |

Total ~5-8 days of dev work, ~$0.50 cascade spend, no infrastructure
additions, no breaking changes.

---

## What this research deliberately does NOT recommend

- **Switching to GraphRAG** — silent edge hallucination is the wrong
  failure mode for clinical content.
- **Adopting LiteLLM proxy** — premature for current scale + recent
  supply-chain compromise (Mar 2026) is a real concern.
- **Switching from ChromaDB** — under 1M chunks, no benefit.
- **Replacing the cascade harness with DSPy** — Medii's tier rules are
  intentionally explicit; auto-routing is the wrong abstraction.
- **Running MedGemma 27B locally** — physically cannot fit on 8GB.
- **Running NV-Embed-v2** — non-commercial licence kills it.
- **Pivoting to an online API** — destroys the "wiki is the codebase"
  identity that makes Medii reviewable.
- **Adopting a notebook-driven runtime** — appropriate for
  experimentation, wrong for production.

---

## References

### Knowledge structure
- [Atlan — LLM Wiki vs RAG: The Karpathy Concept and Enterprise Reality](https://atlan.com/know/llm-wiki-vs-rag-knowledge-base/)
- [Plaban Nayak — Beyond RAG: How Karpathy's LLM Wiki Pattern Builds Knowledge That Compounds](https://levelup.gitconnected.com/beyond-rag-how-andrej-karpathys-llm-wiki-pattern-builds-knowledge-that-actually-compounds-31a08528665e)
- [Analytics Vidhya — From Karpathy's LLM Wiki to Graphify](https://www.analyticsvidhya.com/blog/2026/04/graphify-guide/)
- [AI-Chain — 2 Knowledge Graph Paths + 4 Selection Questions](https://ai-chain.tw/en/blog/graphify-vs-karpathy-llm-wiki-knowledge-base-guide/)
- [DAIR.AI — LLM Knowledge Bases](https://academy.dair.ai/blog/llm-knowledge-bases-karpathy)
- [Karpathy LLM Wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)

### PDF→Markdown extraction
- [ThemenOnLab — Best Open-Source PDF-to-Markdown Tools 2026](https://themenonlab.blog/blog/best-open-source-pdf-to-markdown-tools-2026)
- [Jimmy Song — Marker vs MinerU vs MarkItDown deep dive](https://jimmysong.io/blog/pdf-to-markdown-open-source-deep-dive/)
- [GitHub — Marker (datalab-to)](https://github.com/datalab-to/marker)
- [GitHub — MinerU (opendatalab)](https://github.com/opendatalab/mineru)
- [Systenics AI — MarkItDown / Docling / Mistral Document AI deep dive](https://systenics.ai/blog/2025-07-28-pdf-to-markdown-conversion-tools/)

### Cloud LLM landscape
- [OpenRouter Models](https://openrouter.ai/models)
- [costgoat — OpenRouter Pricing Calculator (Apr 2026)](https://costgoat.com/pricing/openrouter)
- [costgoat — LLM API Pricing Comparison (Apr 2026)](https://costgoat.com/compare/llm-api)
- [Digital Applied — LLM API Pricing Index Q2 2026](https://www.digitalapplied.com/blog/llm-api-pricing-index-q2-2026-cost-per-token)
- [BuildFastWithAI — DeepSeek V4-Pro Review](https://www.buildfastwithai.com/blogs/deepseek-v4-pro-review-2026)

### Medical LLMs
- [Google Research — MedGemma](https://research.google/blog/medgemma-our-most-capable-open-models-for-health-ai-development/)
- [arXiv — MedGemma Technical Report](https://arxiv.org/html/2507.05201v1)
- [Nirmitee — Healthcare LLM Landscape 2026](https://nirmitee.io/blog/healthcare-llm-landscape-2026-medgemma-meditron-clinical-model-guide/)
- [HuggingFace — Open Medical-LLM Leaderboard](https://huggingface.co/blog/leaderboard-medicalllm)
- [Picovoice — Medical Language Models 2026](https://picovoice.ai/blog/medical-language-models-guide/)
- [GitHub — Meditron (epfLLM)](https://github.com/epfLLM/meditron)

### Local LLMs
- [LocalLLM.in — Best Local LLMs for 8GB VRAM (2026)](https://localllm.in/blog/best-local-llms-8gb-vram-2025)
- [LocalAIMaster — Best Small AI Models with Ollama 2026](https://localaimaster.com/blog/small-language-models-guide-2026)
- [PromptQuorum — Local LLM Hardware Guide 2026](https://www.promptquorum.com/local-llms/local-llm-hardware-guide-2026)

### Embeddings
- [BentoML — Best Open-Source Embedding Models 2026](https://www.bentoml.com/blog/a-guide-to-open-source-embedding-models)
- [PreMAI — Best Embedding Models for RAG 2026](https://blog.premai.io/best-embedding-models-for-rag-2026-ranked-by-mteb-score-cost-and-self-hosting/)
- [Modal — Top Embedding Models on MTEB](https://modal.com/blog/mteb-leaderboard-article)
- [Awesome Agents — MTEB Rankings March 2026](https://awesomeagents.ai/leaderboards/embedding-model-leaderboard-mteb-march-2026/)

### Vector stores
- [4xxi — Vector Database Comparison 2026](https://4xxi.com/articles/vector-database-comparison/)
- [Encore — Best Vector Databases 2026](https://encore.dev/articles/best-vector-databases)
- [SaltTechno — Vector Database Benchmark 2026](https://www.salttechno.ai/datasets/vector-database-performance-benchmark-2026/)

### Chunking
- [Anthropic — Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval)
- [Jina AI — Late Chunking](https://jina.ai/news/late-chunking-in-long-context-embedding-models/)
- [PMC — Comparative Evaluation of Advanced Chunking for Clinical Decision Support](https://pmc.ncbi.nlm.nih.gov/articles/PMC12649634/)
- [Weaviate — Chunking Strategies for RAG](https://weaviate.io/blog/chunking-strategies-for-rag)
- [Firecrawl — Best Chunking Strategies for RAG 2026](https://www.firecrawl.dev/blog/best-chunking-strategies-rag)

### Cascade orchestration
- [DSPy](https://dspy.ai/)
- [LiteLLM Providers](https://docs.litellm.ai/docs/providers)
- [Trend Micro — LiteLLM Supply Chain Compromise (Mar 2026)](https://www.trendmicro.com/en_us/research/26/c/inside-litellm-supply-chain-compromise.html)
- [aimultiple — LLM Orchestration: Top 22 Frameworks 2026](https://aimultiple.com/llm-orchestration)

### Structured output
- [GitHub — Instructor (567-labs)](https://github.com/567-labs/instructor)
- [Instructor docs](https://python.useinstructor.com/)
- [Pydantic — How to Use Pydantic for LLMs](https://pydantic.dev/articles/llm-intro)
- [DEV — Top 5 Structured Output Libraries 2026](https://dev.to/nebulagg/top-5-structured-output-libraries-for-llms-in-2026-48g0)

### Medical NLP
- [GitHub — MedCAT (CogStack)](https://github.com/CogStack/MedCAT)
- [MedCAT docs](https://medcat.readthedocs.io/en/latest/main.html)
- [ScienceDirect — MedCAT paper](https://www.sciencedirect.com/science/article/abs/pii/S0933365721000762)
- [SNOMED Implementation — NLP](https://www.implementation.snomed.org/natural-language-processing)

### Image rights
- [Wellcome Collection (Public Domain Review)](https://publicdomainreview.org/collections/source/wellcome-collection/)
- [Met Museum — Open Access](https://www.metmuseum.org/about-the-met/policies-and-documents/open-access)
- [Met Museum — Image and Data Resources](https://www.metmuseum.org/policies/image-resources)
