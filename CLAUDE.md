# Medii - AI Agent Instructions

## Project Overview
Offline medical data ingestion pipeline for building a Medical Learning PWA. Converts PDFs, scrapes, and audio into chunked, embedded, quality-checked Markdown with structured anchors.

## Critical Constraints
- **Clinical accuracy is #1.** Never hallucinate data formatting or flatten medical tables.
- **Hardware:** RTX 3070 (8GB VRAM). Never run concurrent major models (layout parsing + embedding simultaneously).
- **Architecture:** Follow `ARCHITECTURE.md` for folder paths. Do not invent new structures.

## Mandatory Workflow
1. **Before coding:** Read `AI_HANDOFF.md` for current state and blockers. Run `git status` and `git diff` to see prior work.
2. **Git discipline:** Commit after each logical block. Never force-add files in `.gitignore` (no PDFs, sqlite, .venv).
3. **After coding:** Update `AI_HANDOFF.md` (brief: current state, errors, next step). **Append to `CHANGELOG.md` after every major component** — the human reads CHANGELOG to see the project's evolution; AI_HANDOFF is for the next AI's context window. If CHANGELOG drifts, the human loses the trail.
4. **Before pushing to GitHub:** the `pre-push` hook (installed via `bash tools/install_hooks.sh`) auto-syncs every `.md` file into the **separate** `Medii_Markdown_Mirror` repo at `~/Documents/Medii_Markdown_Mirror/` and pushes that mirror to its own GitHub remote. If the hook is not installed (fresh clone), run `bash tools/sync_markdown_mirror.sh` manually before `git push`. The mirror exists so notes/specs are accessible from a tablet without cloning the whole project. Failures of the mirror sync are non-blocking — the Medii push still succeeds.
5. **Logging:** Keep `AI_HANDOFF.md` extremely brief to save tokens.

## Markdown Mirror — one-time setup per machine
1. `bash tools/install_hooks.sh` — installs the `pre-push` hook into the shared `.git/hooks/` (worktree-safe).
2. Ensure `~/Documents/Medii_Markdown_Mirror/` exists, is a git repo, and has an `origin` remote pointing at the mirror's GitHub URL. If the directory does not exist, the hook logs a warning and skips the sync.
3. Override the mirror destination by exporting `MEDII_MIRROR_DIR=/some/other/path` before pushing.

## Key Files
- `src/01_ingest.py` - Multi-source ingestion (PDF→MD+images), pause/resume (PyMuPDF4LLM path)
- `src/01b_docling_extract.py` - Docling extractor on RTX 3070 (preserves tables/math)
- `src/02_embed.py` - Chunking + embeddings (BGE-M3 FP16 on cuda:0)
- `src/03_qc_audit.py` - 3-tier quality check (Code→AI→Human)
- `src/04_image_triage.py` - Image classification via Ollama vision
- `src/05_case_drafter.py` - Stage 4: planner/drafter/verifier cascade → MD case
- `src/anchor_manifest.py` - Structured anchor sidecars for provenance
- `src/oversight/` - Cloud-first cascade (cheap → cross-check → deep) via OpenRouter
- `src/wiki/` - Karpathy-style compounding wiki layer (schemas, ingest, audit, migrate). Canonical design at `plans/wiki_layer_integrated.md` (cross-disciplinary sub-corpora + subtype-aware variants); `plans/wiki_layer.md` is the original 8-pitfall proposal (superseded). `plans/oversight_harness.md` is the implemented-then-superseded original cascade design.
- `corpus/wiki/` - Concept-organised wiki pages with sub-corpora (clinical, history, biomechanics, art, nature). Taxonomy at `corpus/wiki/_taxonomy.yaml`; per-sub-corpus voice guides at `corpus/wiki/_style/`.
- `tools/sync_markdown_mirror.sh` - mirror sync script (called by pre-push hook)
- `tools/install_hooks.sh` - installs git hooks from `tools/git-hooks/`
- `tools/gpu_check.py` - wraps any command, snapshots nvidia-smi, pins CUDA_VISIBLE_DEVICES=0
- `tools/bench_embed.py` - BGE-M3 throughput sanity bench
- `corpus/` - Data lifecycle (raw→extracted→chunks→wiki→cases)

## Tech Stack
- Python 3.12 for the heavy ML stack, Python 3.14 for the lightweight cascade scripts (see "Dual venv" below).
- PDF→Markdown: PyMuPDF4LLM (fast, text-only) **or** Docling + Granite-Docling-258M (preserves tables/math/layout — used for image- and table-heavy sources).
- Embeddings: BGE-M3 via sentence-transformers, pinned to `cuda:0` (RTX 3070).
- Vector store: ChromaDB (local, persistent).
- Cloud LLMs via OpenRouter (single API key drives the cascade): DeepSeek V4 Flash (executor), o4-mini (cross-check), DeepSeek V4 Pro (verifier). Anthropic + OpenAI direct as secondary fallback.
- Local Ollama (MedGemma / Qwen) is OPTIONAL fallback only — set `MEDII_OFFLINE=1` to force it.

## Dual venv — important
This project uses **two separate virtual environments** because the heavy ML stack (torch + Docling + sentence-transformers) doesn't have wheels for Python 3.14 yet.

| venv | Python | Purpose | Scripts |
|---|---|---|---|
| `.venv/` | 3.14 | Cascade, drafter, bench, oversight gates | `src/05_case_drafter.py`, `src/oversight/*`, `oversight.bench`, `oversight.gates` |
| `.venv-ml/` | 3.12 | GPU-bound stages | `src/01b_docling_extract.py`, `src/02_embed.py`, `src/04_image_triage.py` (vision) |

**ML venv setup** (one-time per machine):
```bash
# 1. Bootstrap uv (already in .venv via pip install uv)
.venv/Scripts/python.exe -m uv python install 3.12
.venv/Scripts/python.exe -m uv venv .venv-ml --python 3.12

# 2. Bootstrap pip into .venv-ml
.venv-ml/Scripts/python.exe -m ensurepip --upgrade
.venv-ml/Scripts/python.exe -m pip install --upgrade pip

# 3. Install CUDA-enabled torch FIRST (the dedicated index is critical;
#    omitting it gets you a CPU build silently).
.venv-ml/Scripts/python.exe -m pip install \
    --index-url https://download.pytorch.org/whl/cu124 torch torchvision

# 4. Then the rest from requirements-ml.txt
.venv-ml/Scripts/python.exe -m pip install -r requirements-ml.txt

# 5. Verify the 3070 (not the iGPU) is detected.
.venv-ml/Scripts/python.exe -c "import torch; print(torch.cuda.get_device_name(0))"
# expected: NVIDIA GeForce RTX 3070
```

**Per ARCHITECTURE.md, the GPU cannot run concurrent neural workloads on 8 GB VRAM.** Run extract → embed → triage sequentially, never in parallel.

## Creative Leeway
You have autonomy over engineering decisions. If you find a better pattern or library, implement it. Log reasoning in `AI_HANDOFF.md`.
