# AI Shift Handoff Log

> **MANDATORY INSTRUCTION FOR ALL AI AGENTS:**
> Read this file IMMEDIATELY upon starting a session.
> Update it IMMEDIATELY before concluding if you made structural changes.
> Keep it BRIEF — long-form history belongs in `CHANGELOG.md`.

## [Current State — 2026-04-26]

Branch `claude/wiki-layer` (PR #3 open). Cloud-first cascade is wired,
the markdown mirror is hardened, the local ML stack runs on the RTX 3070,
and the wiki layer infrastructure (steps 1-2 of the integrated plan)
has landed. **STEMI prototype attempted and failed at the editor step**
— diagnosis is the immediate next job. The integrated plan
(`plans/wiki_layer_integrated.md`) is the canonical design.

### Cascade pattern (planner-executor-verifier)

Single OpenRouter API key drives three independent providers across
three tiers — single-vendor mistakes can't silently pass:

| Tier | Role | Model (config.yaml) | Price ($/MTok in/out) |
|---|---|---|---|
| 1 — cheap_judge | Bulk executor | `deepseek/deepseek-v4-flash` | 0.14 / 0.28 |
| 2 — bulk_auditor | Cross-check | `z-ai/glm-5.1` (configured) — `openai/o4-mini` is the bake-off-validated reliable JSON producer | 1.05 / 3.50 |
| 3 — deep_critic | Sampled verifier | `deepseek/deepseek-v4-pro` | 1.74 / 3.48 |
| 1' (offline) | Drill mode | Ollama Qwen / MedGemma | 0 (MEDII_OFFLINE=1) |
| 2nd fallback | If OR is down | Anthropic Haiku/Opus + OpenAI mini | (direct vendor) |

Cost shape: ~95% of calls hit Tier 1; Tier 3 only sees flagged + 5%
PASS-audit samples. Pattern follows the 2026 RouteLLM-style cascade
literature.

### Bake-off finding (relevant for the wiki editor failure)

A multi-model bake-off on the QC + ingest seed prompts found:

- **`openai/o4-mini` was the most reliable JSON producer** — never
  returned None on either prompt set, scored 100% on the simple short
  prompts.
- `deepseek/deepseek-v4-flash` and Kimi K2.6 / GLM-5.1 had occasional
  JSON parse failures on harder prompts (the failures were *format*
  failures, not wrong-answer failures).
- V4-Flash was kept as the configured `cheap_judge` for cost (4× cheaper
  than Kimi); o4-mini sits in `bulk_auditor` as the cross-check tier.

When the wiki editor (a long deeply-nested JSON output prompt) returned
None for V4-Flash, this matches the bake-off failure mode exactly.

### Wiki layer infrastructure (steps 1-2 of integrated plan)

Landed in `claude/wiki-layer`:

- `corpus/wiki/_taxonomy.yaml` — 5 clinical concepts seeded with allowed
  variant IDs (`stemi: [anterior, inferior, lateral, posterior, rv]`).
- `corpus/wiki/_style/{clinical,history,biomechanics,art,nature}.md` —
  per-sub-corpus voice + section-order rules.
- `src/wiki/{schemas,ingest,audit,migrate}.py` — Pydantic page schema,
  5-step ingest pipeline, deterministic provenance walker, schema migrations.
- `src/oversight/schemas.py` + `config.yaml` — `WikiPlanVerdict` /
  `WikiEditVerdict` / `WikiContradictionVerdict` shapes; `stages.wiki_ingest`
  + `stages.wiki_audit` blocks; remediation map for new finding codes.

**STEMI prototype run — FAILED.** Editor step returned `None` (cascade
silently discarded a JSON parse failure on V4-Flash for the deeply
nested editor prompt). Cost so far: $0.0015. **Decision gate per the
integrated plan is NOT yet crossed** — until the prototype produces a
wiki page, `05_case_drafter.py` continues to read raw extracted markdown.

### Local ML stack on RTX 3070

`.venv-ml/` (Py 3.12) has torch 2.6.0+cu124, Docling, sentence-transformers,
ChromaDB, reportlab. BGE-M3 in FP16 baseline: **230 chunks/sec**. Docling
+ Granite-Docling-258M auto-downloads on first call; preserves tables /
math / layout where PyMuPDF4LLM flattens them. CUDA pinning via
`CUDA_VISIBLE_DEVICES=0` so the iGPU stays out of compute. See
`CLAUDE.md` → "Dual venv" for setup.

### Markdown mirror

Sibling repo at `~/Documents/Medii_Markdown_Mirror/` mirrors every `.md`
file. Pre-push git hook syncs on every push. **Hardened against env-var
leakage** — earlier the hook accidentally `git add`ed Medii's working
tree under the mirror's CWD because `GIT_DIR`/`GIT_WORK_TREE` were still
pointing at the calling repo. Fix: `unset GIT_DIR GIT_WORK_TREE
GIT_INDEX_FILE GIT_OBJECT_DIRECTORY GIT_COMMON_DIR` + post-cd verification
that `git rev-parse --show-toplevel` is the mirror dir. Bogus commit
4b7ef78 was reverted in 2c4484d. **Do not push without `tools/install_hooks.sh`
having run on the worktree.**

## [Active Blockers]

1. **`OPENROUTER_API_KEY`** must be exported in the shell that runs the
   pipeline. One key drives all three cascade tiers. Until set, every
   `oversight.router.judge_text` call returns `None` and cascade-driven
   stages no-op.
2. **`qrels_seed.yaml expected_chunk_ids`** are still empty — embed
   gate self-skips until backfilled with real chunk IDs from a first
   end-to-end embed run.

## [Next Immediate Step]

Part A (markdown harmonisation) and Part B (debug logging + balanced-
brace JSON parse + cheap_judge default fix) of `plans/wiki_layer_integrated.md`
have **landed** on `claude/wiki-layer` (commits 141e00f + 6865873;
PR #3 updated). The diagnostic infrastructure is in place. The STEMI
rerun (Part C) is the next operator-action step.

### Part C — STEMI prototype rerun (operator runs this)

Requires `OPENROUTER_API_KEY` exported in the shell. Cost: ~$0.005.

```bash
# Windows (PowerShell):
$env:OPENROUTER_API_KEY = "sk-or-..."
$env:MEDII_DEBUG_DIR = "db/cascade_debug"
$env:PYTHONPATH = "src"
.venv/Scripts/python.exe -m wiki.ingest `
    --source "corpus/extracted_md/bmj/BMJ_ST-elevation*.md" `
    --source-id bmj_stemi --concept stemi --max-source-chars 12000

# Bash:
export OPENROUTER_API_KEY=sk-or-...
export MEDII_DEBUG_DIR=db/cascade_debug
export PYTHONPATH=src
.venv/Scripts/python.exe -m wiki.ingest \
    --source "corpus/extracted_md/bmj/BMJ_ST-elevation*.md" \
    --source-id bmj_stemi --concept stemi --max-source-chars 12000
```

Three outcomes:

- **(a) Editor succeeds** — the balanced-brace JSON fix was enough.
  `corpus/wiki/clinical/stemi.md` lands. Run `python -m wiki.audit
  corpus/wiki/clinical/stemi.md` and compare side-by-side with the
  raw extracted markdown.
- **(b) Editor still returns None** — `db/cascade_debug/` now contains
  the raw V4-Flash output + the prompt that triggered it. Inspect:
  - **Truncated mid-JSON?** Bump `max_tokens` for the cheap tier
    (`src/oversight/router.py:120` — currently 1024). Try 4096.
  - **Prose with refusal language?** V4-Flash can't handle this prompt.
    Switch `router.cheap_judge.model` to `openai/o4-mini` in
    `config.yaml` for `wiki_ingest` only (the bake-off proved
    o4-mini is the most reliable JSON producer).
- **(c) Both V4-Flash and o4-mini fail consistently** — fall back to
  remediation #3 from the plan: split the editor prompt into per-section
  calls (one section = one small JSON output).

### Part C2 — Decision gate

Per the integrated plan: compare the resulting wiki page side-by-side
with the raw extracted markdown. **"If the wiki page reads as obviously
better, proceed. If not, kill the project."**

Specific checks:

- `variants:` frontmatter has meaningful entries for ≥2 of
  `[anterior, inferior, lateral, posterior, rv]`.
- `wiki.audit corpus/wiki/clinical/stemi.md` reports 0
  `MISSING_FOOTNOTE` findings.
- No `**----- Start of picture text -----**` OCR-noise blobs.
- Subjective: does the wiki page read better than the raw extracted
  markdown for someone trying to understand STEMI?

### Part D — Conditional next steps

Only if C2 passes the decision gate, per the integrated plan:

- D1: Drafter wiki-mode (`--concept`, `--variant`, `--include-cross-links`
  flags on `src/05_case_drafter.py`).
- D2: Cross-disciplinary seed (Da Vinci heart studies, Corrigan
  aortic regurgitation, cardiac pressure-volume loops).
- D3: Operator triage tools (deferred, low-priority).
- D4: Bulk ingest (deferred, ~$2-3 across all 22 source files).

## [Pointers]

- Master plan: `medical_learning_pwa_1c60f761.plan.md` (append-only).
- Canonical wiki design: `plans/wiki_layer_integrated.md`.
- Original (superseded) wiki design: `plans/wiki_layer.md`.
- Original (superseded) oversight design: `plans/oversight_harness.md`.
- Persistent project history: `CHANGELOG.md`.
- Agent rules: `CLAUDE.md` is the single source of truth; AGENTS.md /
  GEMINI.md / .cursorrules / .clinerules / .windsurfrules /
  .github/copilot-instructions.md are thin pointers.
