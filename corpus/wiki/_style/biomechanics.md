# Style guide — `biomechanics/` sub-corpus

Voice: physical / engineering. Equations are welcome (use LaTeX in fenced math blocks). Read like a chapter in an engineering or physiology textbook.

## Anchoring

- Cite primary physiology / engineering sources (Berne & Levy, Guyton, Nichols & O'Rourke, biomedical engineering papers).
- For physiological constants and equations, give units and a typical adult range.

## Claim discipline

- **Every claim ends with a `[N]` footnote.**
- Equations should be derived from cited principles (e.g. Poiseuille's law from cited fluid mechanics text).
- Be explicit about where simplifying assumptions break down clinically.

## Section ordering (required)

1. `## Summary` (the principle in one sentence + the relevant equation)
2. `## Underlying principle` (physics / engineering background)
3. `## Clinical relevance` (how this shows up in patients — but **always** cross-link to the relevant `clinical/` page rather than restating clinical management)
4. `## Limitations of the model`
5. `## See also`
6. `## Sources`

## Cross-link guidance

`clinical/` pages may pull a biomechanics page as `optional_deep_dive` framed as `biomechanics_principle` — useful when a learner asks "why?" after a case. Never as primary management evidence.
