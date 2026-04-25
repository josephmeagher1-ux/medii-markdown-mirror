# Medii - Cross-Tool Agent Instructions

> This file is read by Antigravity, Cursor, Claude Code, and other AGENTS.md-compatible tools.

## Project Overview
Medii is an offline medical data ingestion pipeline for a Medical Learning PWA. It processes PDFs, web scrapes, and audio into chunked, embedded, quality-checked Markdown with structured provenance anchors.

## Mandatory Rules for All Agents

### Clinical Safety
- Clinical accuracy is the top priority. Never hallucinate medical data or destroy table formatting.
- When uncertain about medical content structure, preserve the original formatting.

### Hardware Constraints
- Target machine: RTX 3070 with 8GB VRAM.
- Never schedule concurrent GPU-heavy operations (layout parsing + embedding).
- Prefer streaming/batched processing over loading full datasets into memory.

### Workflow
1. **Start of session:** Read `AI_HANDOFF.md` for current state. Run `git status` and `git diff`.
2. **During work:** Follow `ARCHITECTURE.md` for folder structure. Do not invent new top-level directories.
3. **End of session:** Update `AI_HANDOFF.md` (keep brief). Append to `CHANGELOG.md` after major components.

### Git Discipline
- Commit after each logical block of work.
- Never commit files matched by `.gitignore` (PDFs, sqlite, .venv, images).
- If something breaks, `git restore` to a working state and log what failed in `AI_HANDOFF.md`.

### Tech Stack
- Python 3.11, PyMuPDF4LLM, sentence-transformers, ChromaDB, medspacy
- Local LLM: Ollama with Gemma 4 (no cloud API calls for inference)
- Anchor sidecars: `*.anchors.json` beside each `*.md` for provenance

### Creative Leeway
Engineering autonomy is encouraged. Better patterns, libraries, or approaches are welcome as long as you log your reasoning in `AI_HANDOFF.md`.
