# Plan: Cross-disciplinary wiki layer + subtype-aware case generation

> **Status:** operator-approved 2026-04-25; steps 1-2 implemented (PR #3 on
> `claude/wiki-layer`); step 3 (STEMI prototype) attempted, editor blocked
> on V4-Flash JSON parse failure (see `AI_HANDOFF.md` for remediation).
> This is the **canonical** wiki layer design; supersedes `wiki_layer.md`.

## Context

Medii currently runs:

```
extracted_md → chunks → vector index → case drafter (Stage 4 cascade)
```

**Three things this design doesn't do well**, which the operator asked
this plan to address:

1. **Knowledge doesn't compound across sources.** When the BMJ STEMI
   guideline and a LITFL case both describe posterior STEMI, the
   pipeline re-discovers the link on every case run. There's no place
   where "what we collectively know about posterior STEMI" lives.
2. **Subtype handling is brittle.** STEMI has anterior / inferior /
   lateral / posterior / RV variants with different ECGs. The current
   case drafter can pick "STEMI" as a topic but can't be asked for
   *"a posterior STEMI"* with the right ECG examples surfaced. The
   master plan's `presentationVariants[]` is **closed-world deterministic
   selection** — variants must be hand-authored; the model is forbidden
   from improvising them at runtime.
3. **Cross-disciplinary content is "trivia lane"-only.** The master plan's
   Tier 8 history/eponyms corpus and `framing: history_eponym` trivia
   cards are great for lightweight asides ("who was Corrigan?"). The
   operator wants more integration: biomechanics, history of medicine,
   art history (Da Vinci's heart drawings), nature, all woven into how
   concepts are taught — not just dangling at the end of a case.

This plan adopts Karpathy's
[LLM Wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
(extending the proposal at `plans/wiki_layer.md`) with:

- **Sub-corpora** for clinical / history / biomechanics / art / nature, cross-linked.
- **Subtype variants as first-class wiki structure**, materialisable into the case drafter.
- **Tight integration with existing master-plan machinery** — reuses `presentationVariants[]`, `trivia_card`, `framing`, `optionalExtensions[]`, anchor manifests — rather than inventing parallel structures.

## Outcome

After this work the operator can:

- Run `python -m wiki.ingest --source corpus/extracted_md/bmj/BMJ_ST-elevation*.md` and a `corpus/wiki/clinical/stemi.md` page is built/updated, with structured variant subsections (`anterior`, `inferior`, `lateral`, `posterior`, `RV`), each carrying its own ECG image references and per-claim citation footnotes.
- Run `python src/05_case_drafter.py --concept stemi --variant posterior --difficulty medium` and the drafter reads the *posterior* variant block (not the whole guideline), produces a posterior-STEMI case with the right ECG, and pulls historical/cultural cross-links into the `optionalExtensions[]` block (Da Vinci heart anatomy if relevant, Corrigan eponym if a related concept).
- Add a non-clinical source (e.g. `corpus/raw_pdfs/davinci_heart_studies.pdf`) and have it flow into `corpus/wiki/art/davinci_heart_anatomy.md`, automatically cross-linked from `corpus/wiki/clinical/cardiac_anatomy.md` via `## See also`.

## Architecture overview

```
                     CURRENT
extracted_md ─► chunks ─► vector index ─► case drafter
                                              │
                                              ▼
                                        case.md + meta.json

                      AFTER THIS PLAN

corpus/raw_pdfs ─► extract ─► extracted_md ─► chunks ─► vector index
                                  │                        │
                                  ▼                        │
                          wiki ingest pipeline             │  (raw quote
                                  │                        │   lookups)
                                  ▼                        ▼
        corpus/wiki/{clinical,history,biomechanics,art,nature}/
                                  │
                                  ▼
                  case drafter (concept + variant aware)
                                  │
                                  ▼
                       case.md + meta.json
                                  │
                                  ▼
              optional optionalExtensions[] from cross-linked
              non-clinical pages (Da Vinci, Corrigan, etc)
```

The wiki sits **between** chunks and the drafter. Chunks + vector index
stay (verbatim quote lookup, RAG fallback). Case drafter learns to read
from wiki pages by default and from chunks only when wiki coverage is
thin.

## Wiki structure

### Sub-corpora layout

```
corpus/wiki/
├── _taxonomy.yaml              # operator-curated allowed concept IDs
├── _schema_version.txt         # for src/wiki/migrate.py bumps
├── clinical/                   # MLA-aligned medical concepts
│   ├── stemi.md
│   ├── nstemi.md
│   ├── pulmonary_embolism.md
│   └── ...
├── history/                    # history of medicine, eponyms, controversies
│   ├── corrigan_aortic_regurgitation.md
│   ├── osler_internal_medicine.md
│   └── ...
├── biomechanics/               # forces, materials, gait, hemodynamics
│   ├── cardiac_pressure_volume_loops.md
│   ├── arterial_compliance.md
│   └── ...
├── art/                        # art history overlap with anatomy / disease
│   ├── davinci_heart_studies.md
│   ├── rembrandt_anatomy_lesson.md
│   └── ...
├── nature/                     # zoology, evolution, ecology relevant to medicine
│   ├── horseshoe_crab_lal_endotoxin.md
│   ├── snake_venom_drug_discovery.md
│   └── ...
└── _contradictions/            # weekly triage queue
    ├── stemi_p2y12_first_line.md
    └── ...
```

### Page schema (Pydantic-validated YAML frontmatter)

```yaml
---
concept_id: stemi                        # must exist in _taxonomy.yaml
sub_corpus: clinical                     # clinical|history|biomechanics|art|nature
schema_version: 1
tags: [cardiology, ecg, mla:cardiovascular_disease]
sources:
  - id: bmj_stemi
    anchor_manifest: corpus/extracted_md/bmj/BMJ_ST-elevation*.anchors.json
  - id: oxford_handbook_emergency_cardiology
    anchor_manifest: ...
last_updated: 2026-04-25
last_verified_by: deepseek/deepseek-v4-flash
contradictions_count: 1
cross_links:                             # references to other wiki pages
  - history/corrigan_aortic_regurgitation
  - art/davinci_heart_studies
  - biomechanics/cardiac_pressure_volume_loops
variants:                                # subtypes the drafter can request
  anterior:
    summary: "LAD occlusion — ST elevation V1-V4"
    discriminators: [V1-V4 ST elevation, anterior leads]
    image_assets: [bmj_stemi_fig_3, litfl_anterior_001]
    sources: [bmj_stemi#anterior, litfl_032]
  inferior:
    summary: "RCA / LCx occlusion — II, III, aVF"
    ...
  posterior:
    summary: "Often missed — V1-V3 ST DEPRESSION (mirror)"
    discriminators: [V1-V3 ST depression, tall R waves V1-V2]
    image_assets: [litfl_posterior_007]
    sources: [bmj_stemi#posterior_variant, litfl_007]
  lateral: { ... }
  rv: { ... }
---

## Definition
ST-elevation myocardial infarction is myocardial cell death caused by complete
atherothrombotic occlusion of a coronary artery [1].

## Diagnostic criteria
... [1]

## Variants
### Anterior STEMI
... [1] [2]

### Posterior STEMI
Often missed because there's no ST elevation on the standard 12-lead — instead
look for ST depression in V1-V3 with tall R waves [1] [3]. Posterior leads
(V7-V9) confirm.

## Management (UK)
... [1]

## Contradictions
- [bmj_stemi] vs [oxford_handbook_emergency_cardiology] — first-line P2Y12 ...

## See also
- [[history/corrigan_aortic_regurgitation]] — historical context for cardiac auscultation
- [[art/davinci_heart_studies]] — early anatomical understanding of valves
- [[biomechanics/cardiac_pressure_volume_loops]] — why STEMI causes cardiogenic shock

## Sources
[1] bmj_stemi, anchors: ed25519:abc123 ed25519:xyz789
[2] litfl_032, anchors: ...
[3] litfl_007, anchors: ...
```

**Key invariants:**

- Every claim ends with `[N]` footnote pointing at source ID + anchor manifest entry. Provenance audit enforces this deterministically.
- `variants:` is a **structured block in the frontmatter**, not free-form prose. Each variant has a stable ID (`anterior`, `posterior`) the case drafter can request via `--variant <id>`.
- `cross_links:` lists wiki pages from other sub-corpora. The drafter can pull these into `optionalExtensions[]` per the master plan's existing mechanism.
- Contradictions are surfaced, not resolved.

## Subtype-aware case generation — how variants flow

The Da Vinci / posterior STEMI example maps onto existing master-plan
structures **without inventing new schema**:

1. **Wiki page `clinical/stemi.md` defines `variants.posterior`** with discriminators, image assets, and source anchors.
2. **Drafter invocation:** `python src/05_case_drafter.py --concept stemi --variant posterior`.
3. **Drafter loads only the posterior variant block** + cited source chunks (not the whole guideline → cheaper, sharper, no OCR-noise pollution).
4. **Drafter materialises `presentationVariants[]`** in the case JSON per master plan — e.g. `posterior-stemi-male-65`, `posterior-stemi-female-72-atypical`. These are still **frozen, reviewed variants**; runtime selection remains closed-world deterministic per the master plan's safety rule.
5. **Drafter pulls `optionalExtensions[]`** from `cross_links:` — if the operator added `art/davinci_heart_studies.md`, the drafter offers it as `optional_deep_dive` after the case (master plan rule). Da Vinci's drawings of the heart valves become an after-case deep-dive option, not core exam content.
6. **Trivia cards for cross-links** carry `framing: history_eponym` (or new values: `framing: art_overlap`, `framing: biomechanics_principle`, `framing: natural_history`). Per master plan, framing controls UI tone and clinical authority.

End-to-end example:

```bash
python src/05_case_drafter.py --concept stemi --variant posterior \
    --include-cross-links art,history --difficulty medium
```

## Pitfalls — consolidated and extended

### From `plans/wiki_layer.md` (still apply)

| # | Pitfall | Mitigation |
|---|---|---|
| 1 | Hallucination drift on LLM-edited pages | Mandatory per-claim `[N]` footnotes + periodic `tools/wiki_audit.py` |
| 2 | Page proliferation | Hard-coded `_taxonomy.yaml` of allowed concept IDs |
| 3 | Re-ingest cost cascade | Step-1 minimal edit plan; track source→page provenance |
| 4 | Contradiction overload | Queue (not blocker); operator weekly triage |
| 5 | Edit race conditions | Serial ingest + per-page lockfile |
| 6 | Provenance loss on summarisation | Drafter forbidden from editing `## Sources` block |
| 7 | Repo bloat | Wiki in same repo (cases reference relative paths); flatten merge history |
| 8 | Schema lock-in | `schema_version` per page + `src/wiki/migrate.py` |

### New for cross-disciplinary integration

| # | Pitfall | Why it bites Medii | Mitigation |
|---|---|---|---|
| 9 | **Authority confusion** between sub-corpora | A "Da Vinci understood circulation" claim must NEVER be presented at the same authority level as "give aspirin 300 mg PO" | Strict `framing` field on every cross-link insertion. Drafter is **forbidden** from quoting non-clinical sub-corpora as clinical authority. Verifier rejects any non-clinical citation in the `## Management` section. |
| 10 | **Variant proliferation** in `variants:` block | Without bounds, the cascade may invent 12 STEMI sub-subtypes | `_taxonomy.yaml` declares allowed variant IDs per concept (`stemi: [anterior, inferior, lateral, posterior, rv]`). Cascade can only populate listed variants. |
| 11 | **Image asset rights** for Da Vinci / Wellcome / etc | Master plan already warns against "anonymous web rips". Cross-disciplinary content multiplies this risk | Each `image_assets:` entry must reference a manifest with `license` field. Pre-flight check rejects any case using image assets without a license entry. |
| 12 | **Cross-link explosion** — every concept eventually links to every other | A naive drafter will pull 30 cross-links into one case | Cap `optionalExtensions[]` at 3 per case in the drafter prompt. Critic flags excess. |
| 13 | **Subtype source under-coverage** — wiki claims posterior STEMI exists but only has 1 source | Critic should flag "variant has < 2 independent sources" | New verifier finding `VARIANT_UNDERCITED`; promotes to `pending_human_review`. |
| 14 | **Sub-corpus drift in tone** — biomechanics page reading like a textbook, history page reading like a Wikipedia article, inconsistent voice | Style guide per sub-corpus (concise YAML in `corpus/wiki/_style/<sub_corpus>.md`) included in drafter prompt for that sub-corpus |
| 15 | **Cascade cost on cross-disciplinary ingest** — adding a Da Vinci PDF triggers updates to 20 clinical pages because everything links to anatomy | Cross-link suggestions are **proposals**, not auto-edits — operator confirms via `tools/wiki_xlink_queue.py` weekly |

## File-by-file changes

### NEW files

| Path | Purpose | Venv |
|---|---|---|
| `src/wiki/__init__.py` | package init |  |
| `src/wiki/schemas.py` | Pydantic models: `WikiPage`, `WikiVariant`, `WikiEditPlan`, `WikiContradictionFinding`, `WikiCrossLinkProposal` | `.venv` |
| `src/wiki/ingest.py` | 5-step ingest pipeline (plan → edit → contradict → audit → commit). Uses `oversight.router.judge_text`. | `.venv` |
| `src/wiki/audit.py` | Deterministic provenance walker (every claim → footnote → anchor manifest entry). Re-runnable on the whole wiki. | `.venv` |
| `src/wiki/migrate.py` | Deterministic schema migrations between `schema_version` bumps. No cascade calls. | `.venv` |
| `src/wiki/cross_links.py` | Cross-link proposal CLI: scans new pages, suggests `cross_links:` additions to other sub-corpora's pages, queues for operator approval. | `.venv` |
| `corpus/wiki/_taxonomy.yaml` | Operator-curated allowed concept IDs + per-concept variant IDs. ~250 entries seeded from MLA condition list. | (data) |
| `corpus/wiki/_schema_version.txt` | Single integer; bumped when `WikiPage` schema changes incompatibly. | (data) |
| `corpus/wiki/_style/<sub_corpus>.md` | Style guides per sub-corpus (clinical: terse + UK practice; history: narrative + dated; etc). | (data) |
| `tools/wiki_audit.py` | Wrapper around `src/wiki/audit.py` for cron-style scheduling. | `.venv` |
| `tools/wiki_xlink_queue.py` | Operator CLI: review pending cross-link proposals, accept/reject. | `.venv` |
| `tools/wiki_contradiction_queue.py` | Operator CLI: weekly contradiction triage. | `.venv` |

### MODIFIED files

| Path | Change |
|---|---|
| `src/oversight/schemas.py` | Add `WikiPlanVerdict`, `WikiEditVerdict`, `WikiContradictionVerdict`, `VariantUndercitedFinding`. (Existing schemas kept; this adds entries.) |
| `src/oversight/config.yaml` | Add `stages.wiki_ingest`, `stages.wiki_audit`. Add `bench.candidates` entry for wiki-page-rewrite-quality benchmarking. |
| `src/05_case_drafter.py` | Add `--concept`, `--variant`, `--include-cross-links` flags. When `--concept` is set, drafter reads `corpus/wiki/clinical/<concept>.md` instead of raw extracted markdown. Pulls `cross_links:` into `optionalExtensions[]` per master plan. **Existing `--source` flag retained** — old workflow still works. |
| `src/anchor_manifest.py` | Add a stable `anchor_id` lookup helper so wiki pages can resolve `[1]` → source path + page + paragraph. (Anchor format already supports this; need lookup utility.) |
| `CLAUDE.md` | Add "Wiki layer" section describing the `corpus/wiki/` layout, the ingest workflow, and the operator weekly triage CLIs. |
| `AI_HANDOFF.md` | Replace the stub "Wiki layer proposal" pointer with "Wiki layer implemented" once the prototype lands. |
| `medical_learning_pwa_1c60f761.plan.md` | Append section explaining how the wiki layer feeds the existing case authoring pipeline (Stage 4 unchanged in spirit, gains a wiki-first input mode). Don't rewrite the master plan — extend it. |

### UNCHANGED files (deliberately)

The following keep working as-is — no rewrite needed:

- `src/01_ingest.py`, `src/01b_docling_extract.py` — extraction is upstream of the wiki, untouched.
- `src/02_embed.py` — chunks + vector index still feed verbatim-quote lookups and RAG fallback.
- `src/03_qc_audit.py` — chunk-level QC still relevant.
- `src/04_image_triage.py` — image triage upstream of `image_assets` linking.
- `src/oversight/router.py`, `frontier.py`, `gates.py`, `bench.py` — cascade is reused as-is.

This is the "rewrite all the code" request interpreted honestly: **extend, don't bulldoze**. The cascade + ML stack work fine. What the project is missing is the wiki layer + cross-disciplinary integration. Adding those is ~12 new files and ~5 small edits, not a 30-file rewrite.

## Existing utilities to reuse (do NOT reinvent)

- `oversight.router.judge_text(prompt, schema, stage, tier)` — every cascade call in wiki ingest goes through this. (`src/oversight/router.py`)
- `oversight.budget.BudgetLedger` — budget cap for wiki ingest cycles.
- `oversight.schemas.*Verdict` — Pydantic patterns to follow for new wiki verdicts.
- `oversight.gates.run_gate("wiki_ingest", samples, ...)` — wiki ingest plugs into the existing gate orchestrator; no parallel orchestrator needed. The new `stages.wiki_ingest` config block declares the local + frontier judges.
- `anchor_manifest.AnchorManifest` — anchor lookup; wiki audit uses this to resolve `[N]` → source path + page.
- Existing `bench.py` — drop in a `wiki-page-rewrite-quality` task to A/B drafters on the same plan + source.
- The pre-push markdown mirror hook auto-syncs `corpus/wiki/**/*.md` to the [medii-markdown-mirror repo](https://github.com/josephmeagher1-ux/medii-markdown-mirror) — no new mirror plumbing needed.

## Rollout order

1. **Bootstrap data only** (1 commit): `corpus/wiki/_taxonomy.yaml` seeded with 5 concepts (`stemi`, `nstemi`, `pulmonary_embolism`, `acute_asthma`, `dka`), 3 sub-corpora skeletons, 5 style guides. ~$0 cascade cost. **DONE in `c7743cd`.**
2. **Wiki ingest pipeline + schemas** (1 commit): `src/wiki/{schemas,ingest,audit,migrate}.py`. Tests on dummy fixtures only — no API calls. ~$0. **DONE in `1dc347c`.**
3. **STEMI prototype** (1 commit + 1 cascade run): ingest the BMJ STEMI guideline → `corpus/wiki/clinical/stemi.md` with all 5 variants populated. Compare side-by-side with the raw extracted markdown. **Decision gate: if the wiki page reads as obviously better, proceed. If not, kill the project.** ~$0.50. **ATTEMPTED — editor returned None on V4-Flash; diagnosis is the next step (see AI_HANDOFF.md).**
4. **Drafter wiki-mode** (1 commit + 1 cascade run): `src/05_case_drafter.py --concept stemi --variant posterior` end-to-end. ~$0.05.
5. **Cross-disciplinary seed** (1 commit + 1 cascade run): hand-draft `corpus/wiki/art/davinci_heart_studies.md`, `corpus/wiki/history/corrigan_aortic_regurgitation.md`, `corpus/wiki/biomechanics/cardiac_pressure_volume_loops.md`. Add `cross_links:` to `stemi.md`. Drafter run with `--include-cross-links art,history` → case has Da Vinci `optionalExtension`. ~$0.05.
6. **Audit + contradiction queue tools** (1 commit): `tools/wiki_audit.py`, `tools/wiki_contradiction_queue.py`, `tools/wiki_xlink_queue.py`. ~$0.
7. **Bulk ingest** (1 commit + 1 large cascade run): all 22 existing extracted markdown files → ~30-50 wiki pages. ~$2-3 cascade cost. Operator triages contradictions in week-1 follow-up.

Total estimated effort: **~3-4 days dev + ~$3 cascade spend**. Each step is independently mergeable.

## Verification

End-to-end test after each commit:

| Stage | Command | Expected |
|---|---|---|
| After step 2 | `.venv/Scripts/python.exe -m wiki.audit --schema-only` | "no schema violations in 0 pages" (wiki empty) |
| After step 3 | open `corpus/wiki/clinical/stemi.md` and compare structure to the schema in this plan; run `.venv/Scripts/python.exe -m wiki.audit corpus/wiki/clinical/stemi.md` | every claim has a `[N]` footnote; every footnote resolves to an anchor in the manifest |
| After step 4 | `python src/05_case_drafter.py --concept stemi --variant posterior --difficulty medium` | case in `corpus/cases/` mentions ST depression V1-V3 + tall R waves V1-V2 (posterior STEMI hallmarks); meta.json shows `wiki_page_used: corpus/wiki/clinical/stemi.md`, `wiki_variant: posterior` |
| After step 5 | `python src/05_case_drafter.py --concept stemi --variant posterior --include-cross-links art,history` | `optionalExtensions[]` in case includes a Da Vinci or Corrigan `optional_deep_dive` with `framing` set correctly |
| After step 6 | `python tools/wiki_audit.py --check-all` | all wiki pages pass per-claim provenance audit; contradictions queue lists known disagreements |
| After step 7 | `python tools/wiki_audit.py --check-all --strict` and operator review of `corpus/wiki/_contradictions/` | < 5 unaudited claims across whole wiki; contradictions queue manageable (< 20 open items) |

GPU constraint: wiki ingest is **CPU-cascade-only** (no torch involved), so it runs in `.venv` and does not contend with the `.venv-ml` GPU work per the ARCHITECTURE.md sequential-neural-workload rule.

## Honest non-goals

- **Not** rewriting the cascade, oversight harness, or ML stack — they work and are validated.
- **Not** auto-generating cross-disciplinary content from scratch — the operator must add source material (PDFs, scans, web content with rights). The wiki layer organises and cross-links what the operator has approved into the corpus, never invents historical or art content from thin air.
- **Not** changing runtime case selection. `presentationVariants[]` still resolves deterministically at PWA runtime per master plan safety rule. The wiki layer feeds **authoring**, not playback.
- **Not** adopting a graph database, Obsidian, Dendron, or any external tool. Wiki pages are plain markdown in git, parseable by any editor. Karpathy's slogan: "the wiki is the codebase."
