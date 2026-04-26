# Project Architecture: Medii PWA Data Pipeline

Offline-leaning medical data pipeline for a Medical Learning PWA. Heavy
neural work runs locally on the RTX 3070; LLM judgement runs in the cloud
through OpenRouter (cheap → cross-check → deep cascade) with local Ollama
as an OPTIONAL fallback.

## Directory Structure

```
Medii/
├── .venv/                      # Py 3.14 — cascade, drafter, oversight (gitignored)
├── .venv-ml/                   # Py 3.12 — torch+CUDA, Docling, BGE-M3 (gitignored)
├── corpus/                     # The data lifecycle
│   ├── raw_pdfs/               # Input PDFs dropped by user (gitignored)
│   ├── raw_audio/              # Input
│   ├── extracted_md/           # PyMuPDF4LLM / Docling output (.md + images/)
│   │   ├── books/              # Textbook markdown + images/
│   │   ├── bmj/                # BMJ guideline markdown + images/
│   │   └── litfl/              # LITFL ECG case markdown
│   ├── wiki/                   # Karpathy-style compounded knowledge
│   │   ├── _taxonomy.yaml      # operator-curated allowed concept IDs
│   │   ├── _schema_version.txt # bumped by src/wiki/migrate.py
│   │   ├── _style/<sub>.md     # voice + section-order guides per sub-corpus
│   │   ├── clinical/           # MLA-aligned medical concepts
│   │   ├── history/            # history of medicine, eponyms, controversies
│   │   ├── biomechanics/       # forces, hemodynamics, materials
│   │   ├── art/                # art history overlap with anatomy / disease
│   │   ├── nature/             # zoology, evolution, ecology relevant to medicine
│   │   └── _contradictions/    # weekly triage queue
│   └── cases/                  # Stage-4 drafter output (.case.md + .meta.json)
├── db/                         # All gitignored
│   ├── vector_index/           # ChromaDB
│   ├── metadata.sqlite         # chunk → page provenance
│   ├── frontier_cost_log.sqlite  # budget ledger
│   ├── review_queue.jsonl      # human review queue
│   ├── qrels_baseline.json     # recall@K baseline
│   └── cascade_debug/          # optional MEDII_DEBUG_DIR raw-response logs
├── plans/                      # Design docs (versioned)
│   ├── oversight_harness.md    # implemented 2026-04-15; superseded by cloud cascade
│   ├── wiki_layer.md           # superseded by wiki_layer_integrated.md
│   └── wiki_layer_integrated.md  # canonical wiki layer + cross-disciplinary plan
├── src/
│   ├── 01_ingest.py            # Multi-source ingest (PDF→MD), pause/resume (PyMuPDF4LLM)
│   ├── 01b_docling_extract.py  # Docling extractor on RTX 3070 (preserves tables/math)
│   ├── 02_embed.py             # Chunk + BGE-M3 FP16 → ChromaDB on cuda:0
│   ├── 03_qc_audit.py          # 3-tier QC (Code → AI → Human)
│   ├── 04_image_triage.py      # MedGemma vision + Opus tie-break for grey-zone
│   ├── 05_case_drafter.py      # Stage 4 — planner/drafter/verifier cascade → MD case
│   ├── anchor_manifest.py      # Structured anchor sidecars for provenance
│   ├── oversight/              # Cloud-first cascade harness
│   │   ├── router.py           # judge_text / judge_vision dispatch
│   │   ├── frontier.py         # OpenRouter + Anthropic + OpenAI clients
│   │   ├── gates.py            # run_gate(stage, samples, ref) orchestrator
│   │   ├── budget.py           # SQLite ledger + hard-cap BudgetExceeded
│   │   ├── adversarial.py      # perturbation + paired-judge transforms
│   │   ├── remediation.py      # finding-code → strategy map
│   │   ├── local_judge.py      # OPTIONAL Ollama fallback
│   │   ├── qrels.py            # recall@K harness for embed gate
│   │   ├── bench.py            # multi-model bake-off
│   │   ├── schemas.py          # Pydantic verdict shapes
│   │   ├── config.py / config.yaml  # tiers, prices, sample sizes, stage configs
│   │   └── seed_samples/       # poisoned regression fixtures
│   └── wiki/                   # Wiki layer (per plans/wiki_layer_integrated.md)
│       ├── schemas.py          # WikiPage, WikiVariant, WikiEditPlan
│       ├── ingest.py           # 5-step PLANNER → EDITOR → CONTRADICT → AUDIT → COMMIT
│       ├── audit.py            # deterministic per-claim provenance walker
│       └── migrate.py          # schema-version migrations (no LLM calls)
├── tools/
│   ├── git-hooks/pre-push      # markdown mirror sync hook source
│   ├── install_hooks.sh        # worktree-safe hook installer
│   ├── sync_markdown_mirror.sh # mirror sync (called by pre-push)
│   ├── gpu_check.py            # CUDA_VISIBLE_DEVICES=0 wrapper, snapshots nvidia-smi
│   └── bench_embed.py          # BGE-M3 throughput sanity bench
├── AGENTS.md / GEMINI.md       # thin pointers → CLAUDE.md (single source of truth)
├── .cursorrules / .clinerules  # thin pointers → CLAUDE.md
├── .windsurfrules / .github/copilot-instructions.md  # thin pointers → CLAUDE.md
├── AI_HANDOFF.md               # current state + active blockers (brief)
├── CHANGELOG.md                # persistent human-readable history
├── CLAUDE.md                   # canonical agent rules
├── ARCHITECTURE.md             # this file
├── PIPELINE_PLATFORM.md        # platform-level deployment notes
├── medical_learning_pwa_1c60f761.plan.md  # master plan (append-only)
├── requirements.txt            # .venv (Py 3.14) — cascade deps
├── requirements-ml.txt         # .venv-ml (Py 3.12) — torch + Docling + ChromaDB
└── setup_env.ps1               # initial setup script
```

## System Constraints

- **Hardware:** RTX 3070 (8GB VRAM). The iGPU drives the display; the
  3070 is reserved for ML work via `CUDA_VISIBLE_DEVICES=0`.
- **Sequential-only neural workloads.** Extract → embed → triage runs
  one at a time. Concurrent layout-parser + embedder will OOM the 3070.
- **Dual venv.** Python 3.14 has no torch/Docling wheels yet, so the
  ML stack lives in a separate `.venv-ml/` (Py 3.12). Cascade and
  drafter scripts use the system `.venv/` (Py 3.14). See `CLAUDE.md`
  → "Dual venv" for setup.
- **Cloud-first cascade.** Judge-grade LLMs run in the cloud via
  OpenRouter (single API key, three independent providers across the
  three cascade tiers — see `src/oversight/router.py` and
  `src/oversight/config.yaml router:`). Local Ollama is an OPTIONAL
  fallback (`MEDII_OFFLINE=1`).
- **Embeddings stay local.** BGE-M3 FP16 on the 3070 — fits cleanly
  in 8 GB when nothing else is on the GPU. ~230 chunks/sec baseline.
- **Wiki-first authoring (when the decision gate passes).** Stage 4
  case drafter reads from `corpus/wiki/clinical/<concept>.md` instead
  of raw extracted markdown. Wiki ingest is a 5-step cascade pipeline
  per `plans/wiki_layer_integrated.md`.

## Conventions

- **Anchor sidecars** (`*.anchors.json`) live next to each `*.md` for
  per-claim provenance. Wiki pages cite source IDs which resolve back
  through anchor manifests to source path + page + paragraph.
- **Markdown mirror.** Every `.md` in this repo is mirrored to a
  sibling repo at `~/Documents/Medii_Markdown_Mirror/` via the pre-push
  hook (`tools/git-hooks/pre-push`). The mirror exists so notes are
  readable from a tablet without cloning the full pipeline.
- **No new top-level dirs without updating this file.** If you add a
  directory at the project root, add it here in the same commit.
