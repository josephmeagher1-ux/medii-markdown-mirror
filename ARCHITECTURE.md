# Project Architecture: Medii PWA Data Pipeline

This is an offline, local data ingestion pipeline for building an accurate Medical Learning PWA.

## Directory Structure
```
Medii/
├── .venv/                  # Excluded from git
├── corpus/                 # The core data lifecycle
│   ├── raw_pdfs/           # Input: PDFs dropped by user (Excluded from git)
│   ├── raw_audio/          # Input
│   └── extracted_md/       # Output: Markdown + Images from PyMuPDF4LLM
│       ├── books/          # Textbook markdown + images/
│       ├── bmj/            # BMJ guideline markdown + images/
│       └── litfl/          # LITFL ECG case markdown
├── db/                     
│   ├── vector_index/       # Chroma/FAISS embeddings
│   └── metadata.sqlite     # Tracks text chunk -> pdf page source
├── src/                    # The processing logic
│   ├── 01_ingest.py        # Connects to Docling/Unstructured.io to output Markdown
│   ├── 02_embed.py         # Chunks MD hierarchically, runs medspacy NER, generates vectors
│   └── 03_qc_audit.py      # Runs the 3-Tier Quality check (Code -> AI -> Human Queue)
├── AI_HANDOFF.md           # The shift-log for AI editors
├── ARCHITECTURE.md         # This file
├── setup_env.ps1           # Initial setup script
└── requirements.txt        # Python dependency list
```

## System Constraints
- **Hardware:** User is running on a local machine with an RTX 3070 (8GB VRAM).
- **Execution Policy:** Neural networks (Embedding models, Layout Parsers, local LLMs) MUST run *sequentially*, never concurrently, to prevent Out-Of-Memory (OOM) failures.
