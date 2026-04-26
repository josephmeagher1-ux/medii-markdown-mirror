# Medii — Cross-Tool Agent Instructions

> **This file used to duplicate (and disagreed with) the rules in CLAUDE.md.**
> **It is now a thin pointer. The canonical source is `CLAUDE.md`.**

## Read first

1. **`CLAUDE.md`** — current rules, dual-venv setup, key files, tech stack.
2. **`AI_HANDOFF.md`** — current state, active blockers, next step.
3. **`ARCHITECTURE.md`** — directory layout, hardware constraints.
4. **`plans/wiki_layer_integrated.md`** — canonical wiki layer design.
5. **`CHANGELOG.md`** — persistent project history.

## Key invariants (non-negotiable)

- Clinical accuracy is #1. Never hallucinate medical content or flatten tables.
- RTX 3070 (8 GB VRAM) is reserved for ML work via `CUDA_VISIBLE_DEVICES=0`. The iGPU drives the display. Run extract → embed → triage **sequentially**.
- **Dual venv:** `.venv/` (Py 3.14) for cascade + drafter; `.venv-ml/` (Py 3.12) for torch + Docling + ChromaDB. See `CLAUDE.md`.
- **Cloud-first cascade** through OpenRouter (`OPENROUTER_API_KEY`). Local Ollama is OPTIONAL fallback (`MEDII_OFFLINE=1`).
- Read `AI_HANDOFF.md` before coding. Update it before concluding if you made structural changes. Append to `CHANGELOG.md` after major components.
- Commit after each logical block. Never force-add gitignored files (PDFs, sqlite, `.venv*`, images).
- Markdown mirror: `bash tools/install_hooks.sh` once per machine; the pre-push hook auto-syncs `.md` files to `~/Documents/Medii_Markdown_Mirror/`.

## Creative leeway

You have autonomy over engineering decisions. Better libraries, patterns, or approaches are welcome — log reasoning in `AI_HANDOFF.md`.
