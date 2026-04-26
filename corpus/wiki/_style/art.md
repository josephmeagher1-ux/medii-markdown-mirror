# Style guide — `art/` sub-corpus

Voice: art-historical, descriptive. Read like a museum catalogue entry crossed with a medical observation.

## Anchoring

- Cite the work's catalogue entry (Royal Collection, Louvre, Wellcome, Met, etc.) with accession number where possible.
- Distinguish what the artist **observed** from what they **inferred** (Da Vinci dissected human hearts but interpreted some structures incorrectly by modern standards — say so).
- **Image rights**: every `image_assets:` entry must have a `license` field. Public-domain pre-1900 works are usually safe; modern works need explicit permission.

## Claim discipline

- **Every claim ends with a `[N]` footnote.** Same rule as everywhere else.
- When an artwork visualises an anatomical structure, the structural claim cites both the artwork and a modern anatomy source.
- Aesthetic claims ("masterful chiaroscuro") need a curatorial source, not just the page author's opinion.

## Section ordering (required)

1. `## Summary` (work, artist, year, medium, where held)
2. `## Anatomical / medical content` (what the work depicts)
3. `## What the artist got right / wrong by modern understanding` (heavily caveated — modern hindsight is cheap)
4. `## Cultural context` (what was known at the time)
5. `## See also`
6. `## Sources`

## Cross-link guidance

`clinical/` pages may pull an `art/` page as `optional_deep_dive` framed as `art_overlap` — e.g. Da Vinci's heart drawings after a STEMI / cardiac anatomy case. **Never** as evidence for diagnosis or management.
