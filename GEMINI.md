# Medii - Antigravity Agent Instructions

## Project
Offline medical data ingestion pipeline for a Medical Learning PWA. Converts PDFs, web scrapes, and audio into chunked, embedded, quality-checked Markdown with structured provenance anchors.

## Critical Rules
- **Clinical accuracy is #1.** Never hallucinate data formatting or flatten medical tables.
- **Hardware:** RTX 3070 (8GB VRAM). Never run concurrent GPU-heavy models (e.g., layout parsing + embedding at the same time).
- **Architecture:** Follow `ARCHITECTURE.md` for all folder paths. Do not create new top-level directories.
- **Before coding:** Read `AI_HANDOFF.md` for current state and blockers. Run `git status` and `git diff`.
- **After coding:** Update `AI_HANDOFF.md` briefly (state, errors, next step). Append to `CHANGELOG.md` after major work.
- **Git:** Commit after each logical block. Never force-add `.gitignore`d files (PDFs, sqlite, .venv).
- **Local-only inference:** All LLM calls go through Ollama (Gemma 4). No cloud API calls for model inference.

## Key Pipeline
`01_ingest.py` (PDF→MD+images) → `02_embed.py` (chunk+embed) → `03_qc_audit.py` (quality check) → `04_image_triage.py` (vision classification)

## Creative Leeway
You have autonomy over engineering decisions. Better patterns, libraries, or approaches are welcome. Log reasoning in `AI_HANDOFF.md`.
