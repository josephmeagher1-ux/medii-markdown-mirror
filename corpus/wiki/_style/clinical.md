# Style guide — `clinical/` sub-corpus

Voice: terse, declarative, examination-ready. Aim for the register of NICE / BMJ Best Practice rather than a textbook.

## Anchoring

- **UK clinical practice is the default.** Use UK drug names, doses, units (mg, mmol/L), and pathways (NICE / BMJ).
- For Republic of Ireland-specific items (immunisation schedule, cancer screening, legal jurisdiction), use Irish sources and tag the section accordingly.
- Never cite US-only management as if it were UK practice.

## Claim discipline

- **Every claim ends with a `[N]` footnote** pointing at a source ID. No exceptions.
- Use absolute statements only when the source supports them. Otherwise hedge ("usually", "in most patients").
- If two sources disagree, surface the disagreement in the `## Contradictions` section. **Never silently pick a winner.**

## Section ordering (required)

1. `## Definition`
2. `## Diagnostic criteria`
3. `## Variants` (with `### Anterior`, `### Inferior`, etc per `_taxonomy.yaml`)
4. `## Management (UK)`
5. `## Contradictions`
6. `## See also` (cross-links to other wiki pages)
7. `## Sources`

## What NOT to include

- Patient narratives — those belong in cases, not wiki pages.
- Speculation, "in my opinion", "experts believe" without a citation.
- Doses or thresholds without an explicit source.
- Drug brand names (use generics).
