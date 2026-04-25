# Medii — Markdown Mirror

This repo is an **auto-generated read-only mirror** of every `.md` file
in the [Medii](https://github.com/) project, preserving the source
directory structure under the project root.

**Do not edit files here directly.** Any change here is overwritten on
the next sync. Edit the source `.md` files in the Medii repo; they
flow into this mirror automatically.

## How it works

A `pre-push` git hook in the Medii repo (`.git/hooks/pre-push`) calls
`tools/sync_markdown_mirror.sh`. Each Medii push:

1. Copies every tracked `.md` file from the Medii working tree into
   this mirror, preserving paths.
2. Removes any `.md` file in the mirror that no longer exists in Medii.
3. Stages, commits (`mirror: sync from medii @ <sha>`), and pushes.
4. Then lets the original Medii push proceed.

If pushing the mirror fails (no remote configured, network down, etc.),
the script logs a warning and lets the Medii push continue regardless —
the mirror is best-effort, not a blocker.

## Excluded paths

- `.venv/`, `.git/`, `node_modules/`, `__pycache__/`
- `.claude/worktrees/` (transient working copies)
- Any path the source repo's `.gitignore` excludes
