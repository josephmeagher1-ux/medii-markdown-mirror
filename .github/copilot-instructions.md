# GitHub Copilot Instructions

## Project: Medii
Offline medical data ingestion pipeline for a Medical Learning PWA. Processes PDFs, web scrapes, and audio into chunked, embedded, QC'd Markdown with structured provenance anchors.

## Rules
- Clinical accuracy is the top priority. Never hallucinate medical data formatting or destroy table structure.
- Hardware constraint: RTX 3070 (8GB VRAM). Never schedule concurrent GPU-heavy models.
- Follow `ARCHITECTURE.md` for all folder paths and structure.
- Read `AI_HANDOFF.md` for current project state before making changes.
- Commit after each logical block. Never commit files matched by `.gitignore`.
- Update `AI_HANDOFF.md` after structural changes. Append to `CHANGELOG.md` after major work.

## Key Patterns
- Ingestion pipeline: `01_ingest.py` → `02_embed.py` → `03_qc_audit.py`
- Anchor sidecars: `*.anchors.json` beside each `*.md` for provenance tracking
- Pause/resume via `corpus/.ingest_progress.json`
- Image extraction at 200dpi PNG via PyMuPDF4LLM
- Local LLM inference only (Ollama with Gemma 4), no cloud API calls
