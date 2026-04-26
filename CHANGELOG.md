# Project History & Changelog

This document is a persistent historical record of the project for the human user. 
Unlike `AI_HANDOFF.md` (which only stores the *current* state to save context window tokens), this file tracks the chronological evolution of the PWA and the work completed by various AI agents over time.

## Initial Setup (2026-04-07)
*   **Architecture Scaffolded:** Drafted the `medical_learning_pwa_1c60f761.plan.md` plan and updated it to enforce a robust **hybrid extraction RAG pipeline** using local offline processing on an 8GB RTX 3070.
*   **Workspace Initialized:** Created the directory structure (`corpus/raw_pdfs`, `db`, `src`) and the dependency setup script (`setup_env.ps1`). 
*   **Quality Control Mapped:** Engineered a 3-tier QC engine (`src/03_qc_audit.py`) featuring Code Heuristics, AI Spot-Checking, and Human Markdown Queues.
*   **AI-Agnostic Framework Locked:** Established local Git tracking and universal `.cursorrules` / `.windsurfrules` / `.clinerules` to force AI agents to autonomously handle their own source control and safely hand-off work via `AI_HANDOFF.md`.

## Data Pipeline Execution (2026-04-07)
*   **Multi-Source Ingestion Engine Built (`01_ingest.py`):** Completely rewrote the ingestion script to dynamically route and process the complex datasets from `Resources`. 
*   **Integrated PyMuPDF4LLM:** Hooked up the framework to cleanly extract Markdown (preserving tables/headers) from the 35 local Medical Textbooks and the scrape of BMJ Guidelines, processing exactly 1 PDF at a time sequentially to protect the 8GB RTX 3070 VRAM.
*   **LITFL Case Normalization:** Wrote the parsing logic to collapse `content.txt` from the 137 ECG cases into standard unified Markdown, including absolute paths to the ECG `.png` files for the frontend to render.

## AI Autonomy & Governance (2026-04-07)
*   **Autonomous Git Workflow Enforced:** Updated all AI rule files (`.cursorrules`, `.windsurfrules`, `.clinerules`) to mandate that AI agents run `git add`, `git commit`, `git status`, and `git restore` themselves — the human user does not manage source control.
*   **Creative Leeway Rule Added:** Gave AI agents permission to experiment with better libraries or patterns, provided they log design rationale in `AI_HANDOFF.md`.
*   **CHANGELOG.md Created:** Split logging into two documents: `AI_HANDOFF.md` (brief, current-state-only for AI context windows) and `CHANGELOG.md` (persistent human-readable project history).

## Ingestion Engine Hardening (2026-04-07)
*   **Pause/Resume System:** Rewrote `01_ingest.py` with a JSON progress file (`corpus/.ingest_progress.json`). Pressing `Ctrl+C` saves state instantly; re-running the script resumes from exactly where it left off.
*   **Live GPU Monitoring:** Added `nvidia-smi` polling every 5 files, printing GPU utilisation %, VRAM usage, and temperature inline with the progress bar.
*   **Graceful Signal Handling:** `SIGINT` is caught cleanly — no corrupted half-written files.

## First Successful Test Run (2026-04-07)
*   **Pipeline Validated End-to-End:** Ran `01_ingest.py` in `TEST_MODE` and confirmed:
    *   **BMJ Guidelines** — Excellent extraction quality (117–172KB structured Markdown per guideline, tables and diagnostic criteria perfectly preserved).
    *   **LITFL ECG Cases** — Clean case text with ECG image paths correctly embedded.
    *   **Books (ECG workbook)** — Image-heavy PDFs yield mostly placeholders; text-rich books (Oxford Handbooks, Robbins) expected to extract well on the full run.
*   **Finding:** Image-heavy PDFs will need a separate OCR/vision pass in a future phase.

## Architecture Plan Update (2026-04-07)
*   **Stage 7 — Independent AI Curriculum Auditor:** Formalized in the master plan. After cases are drafted by the Frontier AI and approved, a separate Auditor AI cross-references all published cases against the UK MLA and USMLE content maps, identifies gaps, and automatically feeds missing topics back to the Frontier Drafter in a loop until 100% curriculum coverage is achieved.

## Cross-Cutting Oversight Harness — Scaffolded (2026-04-15)
*   **Plan Adopted (`plans/oversight_harness.md`):** Single oversight layer wraps every stage (`ingest` / `embed` / `triage` / `qc` / future `case-draft`). Flow per stage: local judge (MedGemma 4B / Qwen 2.5 7B via Ollama) → adversarial perturbation pass → sampled frontier audit (Opus 4.6 + GPT-mini bulk tier) → remediation / human review queue.
*   **`src/oversight/` Package Built:** `budget.py` (SQLite ledger, hard-cap `BudgetExceeded`, CLI `--report`/`--simulate`), `local_judge.py` (shared Ollama REST), `frontier.py` (Anthropic + OpenAI clients booking every call through the ledger), `adversarial.py` (typo / synonym / caption-removal / boundary-shift / image downscale-crop + paired critic–defender judges), `remediation.py` (strategy map + JSONL review queue), `gates.py` (`run_gate(stage, samples, reference) → GateVerdict`), `qrels.py` + `qrels_seed.yaml` (golden recall@K harness for the embed stage), `schemas.py` (Pydantic verdict models), `seed_samples/*.jsonl` (poisoned regression fixtures).
*   **Budget Discipline:** £150 / ~$190 hard cap at 80 % warning (override via `MEDII_BUDGET_USD_CAP` env var). Frontier calls are strictly sampled — never looped over the full corpus.
*   **Dependencies Added:** `pyyaml`, `anthropic`, `openai`, `python-dotenv`, `Pillow`; `pydantic` bumped to `>=2`. Runtime artifacts (`db/frontier_cost_log.sqlite`, `db/review_queue.jsonl`, `db/vector_index/`, `db/qrels_baseline.json`) added to `.gitignore`.
*   **Smoke-Tested:** Budget ledger CLI (records $4.50 for 50k/50k Opus call, passes cap check); adversarial transforms (label-preserving on a medical sentence); package imports cleanly without API keys or Ollama.
*   **Pending Wiring (next session):** `01_ingest.py`, `02_embed.py` (also finish BGE-M3 + ChromaDB), `03_qc_audit.py` (Tier 2.5), `04_image_triage.py` (swap Gemma → MedGemma + Opus tie-break) each need one `oversight.run_gate(...)` call site added per `plans/oversight_harness.md`.

## OpenRouter Cascade Wired (2026-04-25, second pass)
*   **Three Independent Providers via One Key:** Cascade re-pointed at OpenRouter using the latest model IDs as of 2026-04-25. Tier 1 (cheap executor): `moonshotai/kimi-k2.6` ($0.60 / $2.80 per MTok, released Apr 20). Tier 2 (cross-check): `z-ai/glm-5.1` ($1.05 / $3.50, Apr 7). Tier 3 (deep verifier): `deepseek/deepseek-v4-pro` ($1.74 / $3.48, released Apr 24 — 1.6T MoE, 1M context). Three different vendors means a single-vendor mistake cannot silently pass the cascade.
*   **Value Lever Documented:** `deepseek/deepseek-v4-flash` is wired into `pricing:` at $0.14 / $0.28 per MTok (~20× cheaper than Kimi K2.6). Operators can swap it into `router.cheap_judge.model` for high-volume runs after drilling against seed samples to confirm quality.
*   **`openrouter_chat()` Added:** New entry point in `frontier.py` uses the existing `openai` SDK with `base_url=https://openrouter.ai/api/v1`. Supports text + multimodal image content (data: URLs), books every call through the budget ledger, optional HTTP-Referer / X-Title headers for OpenRouter analytics. Falls back to prompt-based JSON if a model rejects `response_format`.
*   **Router Cascade with Graceful Degrade:** `router.judge_text` / `router.judge_vision` now prefer OpenRouter when `$OPENROUTER_API_KEY` is set, fall through to Anthropic Haiku/Opus + OpenAI mini direct SDKs if those keys are present, then to offline Ollama as last resort. Vision routing tries the configured cascade model first (Kimi K2.6 + DeepSeek V4 Pro are both multimodal), falls back to Anthropic vision if OpenRouter errors.
*   **Anthropic + OpenAI Direct Path Retained:** Kept as secondary fallback for OpenRouter outages. Pricing entries for Haiku 4.5 / Sonnet 4.6 / Opus 4.7 / gpt-5-mini stay in `config.yaml`.

## Multi-Model Bake-Off (2026-04-25)
*   **Harness Built (`src/oversight/bench.py`):** Runs the same QC + ingest seed prompts against every candidate model in `bench.candidates`. Hand-curated `GROUND_TRUTH` entries for both task types. Outputs comparison table with accuracy, agreement-with-reference, latency, and cost; raw results to `db/bench_results.jsonl`.
*   **Finding 1 — `openai/o4-mini` is the most reliable JSON producer.** Never returned None on either prompt set, scored 100% on the simple short prompts. Promoted to the `bulk_auditor` cross-check tier.
*   **Finding 2 — Kimi K2.6 / GLM-5.1 had occasional JSON parse failures** (75% / 33% raw accuracy on QC prompts), driven by *format* failures, not wrong-answer failures. Both can answer correctly but occasionally wrap the JSON in prose that breaks `_extract_json`.
*   **Finding 3 — `deepseek-v4-flash` is the right cost-vs-capability default for `cheap_judge`.** 4× cheaper than Kimi K2.6 with similar bulk-judge quality on the short prompts. Swapped into `router.cheap_judge.model` in `config.yaml`. Failure mode on long deeply-nested editor prompts is a known risk (later confirmed in the wiki STEMI prototype).
*   **Hardening:** Swapped console output to ASCII markers (`OK` / `??` / `X`) + `PYTHONIOENCODING=utf-8` after Windows console choked on Unicode `✓`/`✗`. Set `PYTHONUNBUFFERED=1` to prevent empty bench output in monitor.

## Stage 4 — Case Drafter (2026-04-25)
*   **`src/05_case_drafter.py` Built:** Planner-drafter-verifier cascade producing `corpus/cases/<slug>.case.md` + `<slug>.meta.json` sidecar with provenance (plan, verifier_v1/v2, cost, elapsed). PLANNER (deep, with `bulk_alt → cheap` fallback chain) → DRAFTER (cheap, JSON-wrapped via `{"markdown": "..."}` to satisfy executor's response_format constraint) → VERIFIER (bulk_alt) → human review queue.
*   **OCR-Noise Stripping:** `extract_section()` removes `**----- Start of picture text -----**<br>` blocks and `![](...)` image refs before passing to the cascade — those artifacts pollute drafter output and tokens.
*   **3-Tier Planner Fallback:** Hit DeepSeek V4 Pro 429 rate limits (Together upstream). Solution:
    ```python
    for tier_attempt in ("deep", "bulk_alt", "cheap"):
        plan = judge_text(plan_prompt, ..., tier=tier_attempt, ...)
        if plan is not None: break
    ```
    Pipeline survives single-vendor rate limits cleanly.
*   **End-to-End Verified on STEMI:** ~$0.014/case, ~30s elapsed.

## Markdown Mirror Infrastructure (2026-04-25)
*   **Sibling Repo at `~/Documents/Medii_Markdown_Mirror/`** holds a 1:1 copy of every `.md` file in this project. Pushed to its own GitHub remote; the operator reads from there on a tablet without cloning the full pipeline.
*   **`tools/sync_markdown_mirror.sh` + Pre-Push Hook:** Mirrors `git ls-files '*.md'` (respects `.gitignore`), prunes deletions, commits with per-call env-var identity (no global git config touched). Fail-soft: hook never blocks the Medii push.
*   **Versioned Hook Source (`tools/git-hooks/pre-push`):** Worktree-safe installer (`tools/install_hooks.sh`) writes into the shared `.git/hooks/` via `git rev-parse --git-common-dir`.
*   **POST-MORTEM: Bogus Commit `4b7ef78`.** First version of the script left `GIT_DIR` / `GIT_WORK_TREE` set when the hook fired. `cd "$MIRROR_DIR"` succeeded but `git add -A` still pointed at the Medii repo via the env vars → 1548 deletions in one commit. **Fix:** `unset GIT_DIR GIT_WORK_TREE GIT_INDEX_FILE GIT_OBJECT_DIRECTORY GIT_COMMON_DIR` + post-cd verification that `git rev-parse --show-toplevel` resolves to the mirror dir, else abort. Recovery: surgical revert in `2c4484d` (no force-push needed). Future agents: never assume git env-vars are clean inside a hook.

## Local ML Stack on RTX 3070 (2026-04-25)
*   **Dual-Venv Split.** `.venv-ml/` (Python 3.12) for torch + Docling + sentence-transformers + ChromaDB + reportlab; `.venv/` (Python 3.14) for cascade and drafter scripts. Python 3.14 has no torch wheels yet, hence the split. Setup commands documented in `CLAUDE.md` → "Dual venv".
*   **CUDA Wheel Pitfall.** `uv pip install --extra-index-url ...pytorch.org/whl/cu124` silently picked CPU torch from PyPI as "newer" than the CUDA wheel. Switched to plain `pip install --index-url https://download.pytorch.org/whl/cu124 torch torchvision` (no extra index) to force CUDA. Documented in CLAUDE.md.
*   **`tools/gpu_check.py`** wraps any command, snapshots `nvidia-smi` before+after, force-sets `CUDA_VISIBLE_DEVICES=0` so the iGPU stays out of compute. Warns if multiple CUDA devices appear. Normpaths the executable for Windows compat (subprocess.run WinError 2 on forward-slash relative paths).
*   **`src/01b_docling_extract.py`** — Docling extractor pinned to `cuda:0` via `AcceleratorOptions(num_threads=4, device=AcceleratorDevice.CUDA)`. Granite-Docling-258M auto-downloads. Test PDF (1 page, 2x2 table, 2 paragraphs) → 2.7s warm, 0.6 GB VRAM, table preserved as proper Markdown (PyMuPDF4LLM flattens tables).
*   **`src/02_embed.py`** — pin device + FP16 (`model.half()` on Ampere sm_86 for ~2× throughput vs FP32). Override with `MEDII_EMBED_DEVICE=cpu` if extract is mid-run. Throughput baseline: **230 chunks/sec** on the 3070, 2,678 chunks in 11.64s on the BMJ corpus. Whole-corpus embed estimated <30 min.
*   **`tools/bench_embed.py`** — 5-file BMJ corpus throughput sanity bench. Reports chunks/sec + VRAM peak + similarity histogram.

## Wiki Layer Infrastructure — Steps 1-2 of Integrated Plan (2026-04-25/26)
*   **Plan Adopted (`plans/wiki_layer_integrated.md`).** Operator-approved cross-disciplinary wiki layer with subtype-aware case generation. Sub-corpora for clinical / history / biomechanics / art / nature; subtype variants as first-class structure (e.g. `stemi.variants.posterior` with own ECG examples + sources); cross-links pulling non-clinical content into `optionalExtensions[]` per master plan. 15 pitfalls catalogued with mitigations.
*   **Step 1 — Bootstrap Data (`c7743cd`):** `corpus/wiki/_taxonomy.yaml` seeded with 5 clinical concepts + variant IDs (`stemi: [anterior, inferior, lateral, posterior, rv]`); `corpus/wiki/_style/{clinical,history,biomechanics,art,nature}.md` per-sub-corpus voice guides; `corpus/wiki/_schema_version.txt` at v1; sub-corpus dir skeletons. Cross-disciplinary subdirs (`history`, `biomechanics`, `art`, `nature`) open for operator-curated additions.
*   **Step 2 — Pipeline Code (`1dc347c`):**
    *   `src/wiki/schemas.py` — Pydantic `WikiPage`, `WikiPageFrontmatter`, `WikiVariant`, `WikiPageEdit`, `WikiEditPlan`, `WikiContradictionFinding`, `WikiCrossLinkProposal` + `parse_page_markdown()` / `page_to_markdown()` round-trip helpers.
    *   `src/wiki/ingest.py` — 5-step pipeline (PLANNER deep → EDITOR cheap → CONTRADICTION CHECK bulk_alt → AUDIT deterministic → COMMIT). Routes through `oversight.router.judge_text` per the cascade. Reuses anchor manifests for `[N]` footnote provenance.
    *   `src/wiki/audit.py` — deterministic per-claim provenance walker. Findings: `MISSING_FOOTNOTE`, `DANGLING_FOOTNOTE`, `UNDECLARED_SOURCE_ID`, `VARIANT_UNDERCITED`, `SCHEMA_VIOLATION`.
    *   `src/wiki/migrate.py` — schema-version migration scaffold (no migrations registered yet; schema at v1).
    *   `src/oversight/schemas.py` — added `WikiPlanVerdict`, `WikiEditVerdict`, `WikiContradictionVerdict`.
    *   `src/oversight/config.yaml` — `stages.wiki_ingest` + `stages.wiki_audit` blocks; remediation map for new finding codes.
*   **Step 3 — STEMI Prototype: ATTEMPTED, FAILED.** First end-to-end ingest run on `BMJ_ST-elevation*.md` crashed at the EDITOR step. Cascade returned `None` — `_extract_json` failed silently on V4-Flash output for the long deeply-nested editor prompt. Same failure mode the bake-off identified for V4-Flash on hard prompts. Cost: $0.0015 (one cascade call). **Decision gate not yet crossed** — wiki layer is infrastructure-only until the editor succeeds and a STEMI page lands for side-by-side comparison.
*   **PR #3 Open** on `claude/wiki-layer` branch. Honest scope: infrastructure complete; STEMI prototype attempted but blocked on V4-Flash JSON parse failure; diagnosis is the next session's first job. Three remediation options recorded in `AI_HANDOFF.md`.

## Markdown Harmonisation (2026-04-26)
*   **Stale Agent-Rule Docs Found + Fixed.** Six files (AGENTS.md, GEMINI.md, .cursorrules, .clinerules, .windsurfrules, .github/copilot-instructions.md) all still said "local LLM only via Ollama, no cloud calls" — the opposite of where the project went on 2026-04-25. Slimmed each to a thin pointer at `CLAUDE.md` (the canonical source). Five sources of rule drift killed in one pass.
*   **`ARCHITECTURE.md` Refreshed.** Old version listed only 3 src/ files (out of 7) and 0 of the new top-level dirs (`src/oversight/`, `src/wiki/`, `corpus/wiki/`, `corpus/cases/`, `tools/`, `.venv-ml/`). New version covers the full current tree + dual-venv constraints + cloud cascade.
*   **`AI_HANDOFF.md` Re-aligned with Reality.** Cascade tier IDs now match `config.yaml` (cheap_judge is `deepseek-v4-flash`, not Kimi). "Next Immediate Step" rewritten to reflect actual current state (bake-off + drafter + mirror + ML stack + wiki infrastructure done; STEMI editor failure + 3 remediation options as the live blocker).
*   **`plans/wiki_layer_integrated.md` Checked In.** Was previously only at `~/.claude/plans/okay-so-what-i-graceful-honey.md` (outside the repo, inaccessible to a fresh clone). Now versioned alongside the original `plans/wiki_layer.md` (marked SUPERSEDED) and `plans/oversight_harness.md` (marked IMPLEMENTED + superseded by cloud cascade).

## Cloud-First Cascade Pivot (2026-04-25)
*   **Hardware Reality Acknowledged:** RTX 3070 (8GB VRAM) is not capable of running judge-grade LLMs reliably. Local Ollama judges demoted to OPTIONAL fallback (`MEDII_OFFLINE=1` drill mode). Embeddings (BGE-M3) remain local — they fit in VRAM cleanly when the GPU isn't shared with another model.
*   **Planner-Executor-Verifier Cascade Adopted:** New `src/oversight/router.py` documents and implements the pattern. Initial wiring used Anthropic Haiku/Opus + OpenAI gpt-5-mini; superseded later the same day by the OpenRouter swap (above) once the operator confirmed the OpenRouter-anchored model picks. Pattern is a direct application of the 2026 LLM-routing literature (RouteLLM-style cascade): ~95% of frontier-quality judgement at a small fraction of frontier cost.
*   **Frontier Module Refactored:** `frontier.py` now exposes `judge_haiku()` (Tier 1 cheap) alongside `critique_opus()` (Tier 3 deep) and `audit_gpt5mini()` (Tier 2 cross-check). Internal `_anthropic_call` helper removes ~70 lines of duplication between Haiku + Opus paths.
*   **Gate Precondition Relaxed:** `gates.run_gate()` no longer hard-pauses when Ollama is unreachable. New rule: at least one cloud API key must be set (or `MEDII_OFFLINE=1` with a running Ollama). All `_*_local()` probe functions in `gates.py` route through `router.judge_*` instead of calling Qwen / MedGemma directly.
*   **No Pipeline-Stage Edits Needed:** `01_ingest.py`, `02_embed.py`, `03_qc_audit.py`, `04_image_triage.py` continue to call `oversight.run_gate(...)` unchanged. The cascade is entirely internal to the `src/oversight/` package — that was the point of the harness layer.
*   **Smoke-Tested:** All 10 touched files `py_compile`-clean. `import oversight` succeeds and exposes the new router entry points.

## Cross-Cutting Oversight Harness — Wired Into Pipeline (2026-04-15)
*   **Ingest Gate (`01_ingest.py`):** `run_ingest_gate(out_dir, collection)` runs after each `process_books` / `process_bmj` / `process_litfl` call. Samples the 8 most-recently-written Markdown files, audits via `oversight.run_gate("ingest", samples)`, prints verdict and per-finding details. Failures do not abort downstream stages (hard stop is at embed/QC where damage is costlier).
*   **Embed Gate + BGE-M3 + ChromaDB (`02_embed.py`):** Added `embed_and_index()` (BGE-M3 via `sentence-transformers`, persisted to ChromaDB at `db/vector_index/`), `_make_retrieve_fn()` for qrels retrieval, and `run_embed_gate()` (fail-closed on recall@K below config threshold). New CLI flags: `--embed`, `--embed-only`, `--skip-gate`. The stub that has been blocking the pipeline since April 7 is now implemented.
*   **QC Tier 2.5 (`03_qc_audit.py`):** New `run_tier_25()` re-audits a sample of Tier-2 PASSes (5 %) + all Tier-2 flags (100 %) via `oversight.frontier.audit_gpt5mini`. If local-vs-frontier disagreement rate exceeds 10 %, emits `LOCAL_JUDGE_MISCALIBRATED` as an error-severity finding. CLI: `--tier 2.5`, `--skip-frontier`.
*   **Image Triage (`04_image_triage.py`):** `DEFAULT_MODEL` bumped from `gemma4:e4b` to `medgemma:4b-it-q4_K_M` (medical-tuned vision). Added `vision_classify_with_escalation()` — REVIEW verdicts tie-break via Opus vision through `oversight.frontier.critique_opus`. Escalation auto-enables when `ANTHROPIC_API_KEY` is set; every call books through the ledger.
*   **Smoke-Tested:** All four edited pipelines `py_compile`-clean. Budget ledger CLI round-trips a $4.50 synthetic call. Adversarial transforms preserve medical sentences under typo / synonym / caption-removal.
