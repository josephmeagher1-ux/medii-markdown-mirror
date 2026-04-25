# Wiki Layer — adopting Karpathy's "LLM Wiki" pattern for Medii

> **Status:** design proposal, not yet implemented.
> **Author:** Claude Opus 4.7 (oversight from operator).
> **Date:** 2026-04-25.

## Why this matters for Medii specifically

Karpathy's [LLM Wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) (April 2026) describes a knowledge structure where an agent **compounds** knowledge across sources rather than re-retrieving them each query. It's optimised for "a few hundred sources on a topic you're going deep on" — which is **exactly** what Medii is doing with UK clinical medicine for the PWA.

The current pipeline does:

  source PDF → extracted_md → chunks → vector index (RAG) → case drafter

Knowledge **does not compound** between sources. Each chunk lives alone. When the BMJ STEMI guideline says "≥2 mm ST elevation in V2-V3 for men >40" and the BMJ NSTEMI guideline says something subtly different about troponin cutoffs, the pipeline has no place to surface that the cardiology corpus is *internally consistent*. The wiki layer fills that gap.

Karpathy's slogan: **"Obsidian is the IDE; the LLM is the programmer; the wiki is the codebase."** For Medii: the **wiki is the corpus** that the case drafter reads from, not the raw `extracted_md`.

## The structural change

```
                         current:
  source.md → chunks → vector index ─────────► case drafter
                                                       │
                         proposed:                     │
  source.md → chunks → vector index                    │
       │           │                                   │
       └─► wiki ingest ─► concept pages ─► case drafter
                  ▲             │
                  │             └─► contradictions log
                  │
            (re-ingest on source update)
```

Wiki sits **between** chunks and the drafter. Chunks + vector index are still useful for "find me a verbatim quote" lookups; the wiki is what the drafter reads to understand the *concept*.

## Page structure (per concept)

```
corpus/wiki/cardiology/stemi.md
─────────────────────────────────
---
concept_id: stemi
tags: [cardiology, ecg, emergency, mla:cardiovascular_disease]
last_updated: 2026-04-25
last_verified_by: deepseek/deepseek-v4-pro
sources: [bmj_stemi, litfl_case_032, oxford_handbook_emergency_cardiology]
contradictions_count: 1
---

## Definition
ST-elevation myocardial infarction is myocardial cell death caused by complete
atherothrombotic occlusion of a coronary artery [1].

## Diagnostic criteria
ECG criteria require new (or increased) and persistent ST-segment elevation
in at least two contiguous leads of:
- ≥1 mm in all leads other than V2-V3 [1]
- V2-V3: ≥2.5 mm in men <40 y, ≥2 mm in men >40 y, ≥1.5 mm in women [1]

## Management (UK)
1. Aspirin 300 mg PO loading dose immediately [1]
2. ...

## Contradictions
- [bmj_stemi] vs [oxford_handbook_emergency_cardiology] — Oxford gives the
  P2Y12 inhibitor as ticagrelor first-line; BMJ allows clopidogrel as
  acceptable in older patients with bleeding risk. Both are current UK
  practice; flag for case-author attention.

## Sources
[1] bmj_stemi.md, anchors: ed25519:abc123 ... ed25519:xyz789
[2] oxford_handbook_emergency_cardiology.md, page 142
```

**Key invariants:**
- Every claim has a `[N]` footnote pointing to a source + anchor (we already produce anchor manifests in `src/anchor_manifest.py`).
- Contradictions are **surfaced, not resolved** — clinical authority lives outside the LLM.
- YAML frontmatter is Pydantic-validated; cascade can't add free-form keys.
- Tags include MLA / USMLE curriculum codes for syllabus-gap reporting (already in the master plan).

## Ingest pipeline (the heart of the wiki layer)

For each new source markdown file:

1. **Plan** (cascade `tier=deep`, ~1 call):
   - Read the source.
   - Output JSON: `{affected_pages: [stemi, nstemi, ...], new_pages: [], contradictions_to_check: [...]}`

2. **Page edits** (cascade `tier=cheap`, ~1 call per affected page):
   - For each `affected_page`: read current page + relevant source chunks → output a unified diff (or full new page).
   - Pydantic-validate the result against the page schema.

3. **Contradiction check** (cascade `tier=bulk_alt`, ~1 call per page edited):
   - Compare new claims to existing claims on the page.
   - If conflict, write to the `## Contradictions` section, do NOT silently overwrite.

4. **Provenance audit** (deterministic, no LLM):
   - Walk the diff. Every new claim must end with a `[N]` footnote.
   - Reject the edit if any claim is unannotated → re-prompt drafter once.

5. **Commit** (deterministic):
   - `git add corpus/wiki/...` + commit with `wiki: ingest <source_id> -> <pages>`
   - Wiki **is** a git repo (the same one as the project, or a sibling — see "Pitfall 7" below).

Rough cost: **5-8 cascade calls per source ingestion**, ~$0.05-$0.10 per typical BMJ guideline at current OpenRouter prices.

## Pitfalls — 8 specific risks + mitigations

| # | Pitfall | Why it bites Medii specifically | Mitigation |
|---|---|---|---|
| **1** | **Hallucination drift** — LLM-edited pages decay from sources over time | Clinical accuracy is invariant #1 in CLAUDE.md. A page that "drifts" toward smoother prose at the cost of dose accuracy can kill a patient. | Per-claim provenance footnotes are mandatory. Periodic `tools/wiki_audit.py` re-checks every claim against its cited source and flags drift. |
| **2** | **Page proliferation** — no topic boundaries → 1000 pages for "STEMI" | UK med curriculum has a finite set of concepts (MLA condition list ≈ 250). | Hard-code a `corpus/wiki/_taxonomy.yaml` with the allowed concept IDs. Cascade can only edit pages whose ID exists in the taxonomy. New page creation requires explicit operator approval. |
| **3** | **Cost on re-ingest cascade** — touching one common page (e.g. "chest pain") costs N calls × every related source | Most BMJ guidelines reference common symptoms. Re-ingesting 1 updated source could trigger 50+ page edits. | Source ingest computes a *minimal* edit plan in step 1 (`affected_pages`) — only pages whose claims actually change. Track source→page provenance so re-ingest of an unchanged source is a no-op. |
| **4** | **Contradiction overload** — every source disagrees with every other on something | British vs Irish vs international medical guidelines differ subtly all the time. | Contradictions section is a queue, not a blocker. Operator triages weekly via a CLI. Drafter is told to "use the most recent UK source unless overridden". |
| **5** | **Edit race conditions** — concurrent ingest of two sources can corrupt a page | If we parallelise, two cascade calls editing `stemi.md` simultaneously last-writer-wins | Serial ingest only. Per-page lockfile in `corpus/wiki/.locks/`. ARCHITECTURE.md already mandates sequential GPU work; extend to wiki edits. |
| **6** | **Provenance loss on summarisation** — when the cascade rewrites a section, footnote IDs may not survive the rewrite | The "real wiki vs LLM wiki" critique on Karpathy's gist comments calls this out specifically. | Step 4's provenance audit rejects any diff that introduces an unannotated claim. Drafter is **forbidden** from editing the `## Sources` block — only step 5 (deterministic) appends to it. |
| **7** | **Repo bloat** — wiki diffs in every commit pollute the Medii repo's git history | The mirror sync hook would also push every wiki edit to the mirror, doubling noise. | Keep `corpus/wiki/` in the Medii repo (so cases reference it by relative path), but add a `.git-attributes` `merge=ours` to flatten wiki history in main-branch merges. **Or** put wiki in its OWN sibling repo (like the markdown mirror) and reference it via submodule. Decide before implementing. |
| **8** | **Wiki schema lock-in** — once 100 pages exist in a given format, schema changes mean rewriting all 100 | Frontmatter shape will iterate; section ordering will iterate. | Migrations: each page records `schema_version: 1`. A `tools/wiki_migrate.py` script bumps versions deterministically (no cascade calls — pure transforms). Cascade only ever sees the current schema version. |

## Implementation order (if we proceed)

1. **`corpus/wiki/_taxonomy.yaml`** — operator-curated list of allowed concept IDs (≈ 250 from MLA list).
2. **`src/oversight/schemas.py`** addition — `WikiPage` Pydantic model + `WikiEditPlan`.
3. **`src/01c_wiki_ingest.py`** — the 5-step pipeline above.
4. **`tools/wiki_audit.py`** — deterministic provenance walker.
5. **`tools/wiki_migrate.py`** — schema migrations (when schema_version ticks).
6. **`src/05_case_drafter.py` update** — switch primary source from raw chunks to relevant wiki pages, with raw-chunk fallback for verbatim quotes.
7. **`tools/wiki_contradiction_queue.py`** — operator CLI for weekly contradiction triage.

Estimated effort: ~2 days dev + ~$5 to ingest the existing 22 .md files into a starter wiki of ~50 pages.

## Whether to do this now

**Argument for:** the case drafter currently re-reads raw extracted markdown every time, which is wasteful, opaque, and hits OCR artifacts every run. A wiki layer would make case-drafting **faster** (read 1 wiki page, not a 4 KB raw chunk), **cheaper** (the cascade does the heavy concept-extraction *once* at ingest, not every case), and **higher quality** (cross-referenced, contradiction-aware).

**Argument against:** the project's working baseline is functional — case drafting at $0.014/case is not a cost crisis. Ingest pipeline costs go up, not down, with the wiki layer added. The 8 pitfalls above are all real and at least 3 (drift, contradictions, race conditions) need careful mitigation before this is safe to use for clinical content.

**Recommendation:** prototype on a **single concept** first — STEMI — using just the BMJ guideline + 2 LITFL cases. ~10 cascade calls, ~$0.50, one weekend. If the prototype page reads better than the raw extracted markdown and the drafter produces a noticeably better case from it, adopt the pattern. If not, kill it. Either way the prototype is small enough that the sunk cost is trivial.

## References

- [karpathy/llm-wiki gist (April 2026)](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
- [Beyond RAG: how Karpathy's LLM Wiki pattern builds knowledge that actually compounds (Plaban Nayak, Apr 2026)](https://levelup.gitconnected.com/beyond-rag-how-andrej-karpathys-llm-wiki-pattern-builds-knowledge-that-actually-compounds-31a08528665e)
- [LLM Wiki by Karpathy: build a compounding knowledge base (DataScienceDojo, Apr 2026)](https://datasciencedojo.com/blog/llm-wiki-tutorial/)
- [What is Karpathy's LLM Wiki? (MindStudio, Apr 2026)](https://www.mindstudio.ai/blog/andrej-karpathy-llm-wiki-knowledge-base-claude-code)
