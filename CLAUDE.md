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
3. **After coding:** Update `AI_HANDOFF.md` (brief: current state, errors, next step). Append to `CHANGELOG.md` after major components.
4. **Logging:** Keep `AI_HANDOFF.md` extremely brief to save tokens.

## Key Files
- `src/01_ingest.py` - Multi-source ingestion (PDF→MD+images), pause/resume
- `src/02_embed.py` - Chunking + embeddings pipeline
- `src/03_qc_audit.py` - 3-tier quality check (Code→AI→Human)
- `src/04_image_triage.py` - Image classification via Ollama vision
- `src/anchor_manifest.py` - Structured anchor sidecars for provenance
- `corpus/` - Data lifecycle (raw→extracted→chunks)

## Tech Stack
- Python 3.11, PyMuPDF4LLM, sentence-transformers, ChromaDB, medspacy, Ollama (Gemma 4)
- Local-only: no cloud API calls for inference

## Creative Leeway
You have autonomy over engineering decisions. If you find a better pattern or library, implement it. Log reasoning in `AI_HANDOFF.md`.
