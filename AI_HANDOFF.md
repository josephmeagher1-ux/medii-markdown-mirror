# AI Shift Handoff Log

> **MANDATORY INSTRUCTION FOR ALL AI AGENTS:**
> Read this file IMMEDIATELY upon starting a session.
> Update it IMMEDIATELY before concluding if you made structural changes.

## [Current State — 2026-04-25]

**Cloud-first pivot complete + OpenRouter cascade wired.** The oversight
harness no longer assumes local LLM-quality inference. The 8GB RTX 3070
is acknowledged as too small for judge-grade models; local Ollama is
now an OPTIONAL fallback (MEDII_OFFLINE=1 drill mode). Embeddings
(BGE-M3) still run locally — they fit in VRAM cleanly when nothing
else is on the GPU.

### Cascade pattern (planner-executor-verifier)

Documented in `src/oversight/router.py`. Three different providers via
a single OpenRouter API key — single-vendor mistakes cannot silently
pass:

  Tier 1 (cheap, bulk)   **moonshotai/kimi-k2.6**       $0.60 / $2.80 per MTok
  Tier 2 (cross-check)   **z-ai/glm-5.1**               $1.05 / $3.50 per MTok
  Tier 3 (deep, sampled) **deepseek/deepseek-v4-pro**   $1.74 / $3.48 per MTok
  Tier 1' (offline only) Ollama Qwen / MedGemma         (MEDII_OFFLINE=1)
  Secondary fallback     Anthropic Haiku/Opus + OpenAI mini (if OR is down)

Cost shape: ~95% of calls land at Tier 1 (Kimi). Tier 3 (DeepSeek V4
Pro) only ever sees flagged + sampled-PASS items (5% calibration
sentinel) — never the full corpus. Pattern follows the 2026 LLM-routing
literature (RouteLLM-style cascade): ~95% of frontier-quality judgement
at a small fraction of frontier cost.

### Value-tuning lever

`deepseek/deepseek-v4-flash` is wired into pricing at $0.14 / $0.28 per
MTok — **20× cheaper than Kimi K2.6**. Swap `router.cheap_judge.model`
in `config.yaml` to `deepseek/deepseek-v4-flash` if cost > capability
becomes the priority. (V4 Flash has 13B active params; quality may dip
on subtle clinical-coherence calls — drill against seed samples first.)

### Files changed this session

- `src/oversight/config.yaml` — `router.*` block now points at OpenRouter
  IDs (kimi-k2.6, glm-5.1, deepseek-v4-pro). Pricing entries added for
  all four OpenRouter models (incl. v4-flash as a value swap option),
  Anthropic + OpenAI fallbacks kept. `openrouter:` block added with
  base_url + analytics headers.
- `src/oversight/router.py` — NEW. `judge_text` / `judge_vision` cascade
  entry points + `is_cloud_available` / `is_offline_mode`. Routes through
  OpenRouter when `$OPENROUTER_API_KEY` is set, falls back to Anthropic /
  OpenAI direct, then offline Ollama.
- `src/oversight/frontier.py` — added `openrouter_chat()` (OpenAI-compatible
  via `openai` SDK with custom base_url, supports text + multimodal image
  content). Kept `judge_haiku()` / `critique_opus()` / `audit_gpt5mini()`
  as secondary fallback path. `_anthropic_call` helper dedupes Haiku/Opus.
- `src/oversight/gates.py` — `_ingest_local`, `_qc_local`, `_triage_local`
  route through `router.judge_*`. Hard "Ollama unreachable → pause"
  replaced with cloud-availability gate that only requires Ollama when
  `MEDII_OFFLINE=1`.
- `src/oversight/local_judge.py` — docstring marks it as optional fallback.
- `src/oversight/__init__.py` — re-exports the new router entry points.

### Smoke tests performed

- `py_compile` clean on all 10 touched files.
- `python -c "import oversight"` succeeds; `judge_text` / `judge_vision`
  symbols resolve. (Config load fails with `pyyaml` missing — see
  blockers below; this is the user's pip install gate, not a regression.)

### Wiki layer proposal — Karpathy LLM-Wiki pattern

Design proposal at `plans/wiki_layer.md` (2026-04-25): adopt Andrej
Karpathy's "LLM Wiki" pattern as a layer between extracted markdown
and the case drafter. Concept-organised pages (`corpus/wiki/`) get
compounded across sources at ingest time, with strict per-claim
provenance footnotes and a contradictions queue for source disagreements.

8 pitfalls catalogued in the plan with mitigations — the four that
matter most for medical content: hallucination drift, contradiction
overload, provenance loss on summarisation, schema lock-in.

**Recommendation in the plan:** prototype on STEMI only first
(~$0.50, one weekend). Don't roll out wiki-wide until the prototype
proves the case drafter produces better output from a wiki page than
from raw extracted markdown.

### ML stack (`.venv-ml`) — local GPU work on the RTX 3070

The heavy ML stack lives in a **separate venv on Python 3.12** because
the system Python is 3.14 (no torch/Docling wheels yet).

  `.venv/`     Py 3.14 — cascade, drafter, bench, oversight gates
  `.venv-ml/`  Py 3.12 — torch 2.6.0+cu124, Docling, sentence-transformers,
                          ChromaDB, reportlab. Targets `cuda:0` only.

Setup steps + verification commands are in `CLAUDE.md` under
"ML venv setup". Key bits:

- **CUDA 12.4 wheels are installed via the dedicated index URL** —
  omitting `--index-url https://download.pytorch.org/whl/cu124` gets
  you a CPU build silently (we hit this; it's a known pitfall).
- **`CUDA_VISIBLE_DEVICES=0` is set in every GPU script's process env**
  so the iGPU stays out of compute. Verified by `tools/gpu_check.py`
  reporting "torch sees 1 device(s)".
- **BGE-M3 runs in FP16** on the 3070 (sm_86 has native tensor cores),
  ~2× throughput vs FP32, ~1.2 GB VRAM at load + ~2.8 GB peak during
  encode. Override with `MEDII_EMBED_DEVICE=cpu` if extract is mid-run.

Throughput baseline (2026-04-25, BMJ corpus, FP16 BGE-M3):
  2,678 chunks / 11.64 s = **230 chunks/sec** on the 3070.
  Whole-corpus embed estimated < 30 min.

Docling smoke test (Granite-Docling-258M auto-downloaded):
  test PDF (1 page, 2x2 table, 2 paragraphs) → 2.7s warm, 0.6 GB VRAM,
  table preserved as proper Markdown (PyMuPDF4LLM flattens tables).

Tools added:
- `tools/gpu_check.py`     — wraps any command, snapshots nvidia-smi
                              before + after, forces CUDA_VISIBLE_DEVICES=0,
                              warns if multiple CUDA devices appear.
- `tools/bench_embed.py`   — 230 chunks/sec sanity bench, run via gpu_check.
- `src/01b_docling_extract.py` — Docling extractor, CUDA-pinned.

### Markdown mirror (separate GitHub repo)

A sibling repo at `~/Documents/Medii_Markdown_Mirror/` holds a 1:1 copy
of every `.md` file in this project, kept in sync by a `pre-push` git
hook. So that a fresh clone gets the hook, the hook source is versioned
at `tools/git-hooks/pre-push`; install with `bash tools/install_hooks.sh`.
The mirror lets the operator read project markdown from a tablet
(via the mirror's GitHub repo) without cloning the full pipeline. See
the **Markdown Mirror** section in `CLAUDE.md` for setup details.

## [Active Blockers / What the user must grant]

1. **`pip install -r requirements.txt`** in the project `.venv`. Already
   listed all needed deps (anthropic, openai, pyyaml, pillow, sentence-
   transformers, chromadb, pymupdf4llm, etc). I cannot run installs into
   the system python without permission.
2. **OpenRouter API key exported** in the shell that runs the pipeline:
   ```
   set OPENROUTER_API_KEY=sk-or-...
   ```
   That single key drives all three cascade tiers. Anthropic / OpenAI
   keys are optional — set them only if you want the secondary fallback
   path to be live (recommended belt-and-braces).
3. **Pricing in `config.yaml` was pulled from OpenRouter on 2026-04-25.**
   Re-confirm at https://openrouter.ai/<model-id> before the first paid
   batch — OpenRouter occasionally re-prices models.
4. **(Optional) Ollama daemon** only needed if you want to drill the
   harness offline (`MEDII_OFFLINE=1`). Not required for normal runs.
5. **`qrels_seed.yaml` `expected_chunk_ids` are empty.** Embed gate
   self-skips until at least one query has real chunk IDs. Backfill
   after the first end-to-end embed run.

## [Next Immediate Step]

Once the user grants 1 + 2 above, run in this order:

1. **Free dry-run (no API spend, no Ollama):**
   ```bash
   PYTHONPATH=src python -m oversight.gates --stage ingest --dry-run
   PYTHONPATH=src python -m oversight.gates --stage qc --dry-run
   PYTHONPATH=src python -m oversight.gates --stage triage --dry-run
   ```
   `--dry-run` skips the frontier escalation, so this only costs the
   Tier-1 Haiku calls against the seed_samples fixtures (~£0.01 total).

2. **Budget-cap drill:**
   ```bash
   set MEDII_BUDGET_USD_CAP=1
   PYTHONPATH=src python -m oversight.gates --stage qc
   ```
   Expect a clean `BudgetExceeded` after a couple of Opus calls.

3. **Full pipeline TEST_MODE end-to-end:**
   `python src/01_ingest.py` → `python src/02_embed.py --embed`
   → `python src/03_qc_audit.py` → `python src/04_image_triage.py --vision`.

4. **Backfill `qrels_seed.yaml`** with real chunk IDs from step 3,
   then re-run `02_embed.py` to baseline recall@K.

5. **Stage 4 — Case drafter.** Harness slot is wired
   (`stages.case_draft.{planner,executor,verifier}` in config). The
   drafter module itself does not yet exist. Pattern is the same:
   Opus 4.7 plans the case → Haiku 4.5 drafts sections → gpt-5-mini
   cross-checks facts → human review queue.
