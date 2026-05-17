# `eval-runs/` — frozen historical archive

This directory contains the validation runs and review prompts produced during the weekend build session (2026-05-16 → 2026-05-17). Files here are **frozen audit trail**, kept for reproducibility and historical reference.

## What lives here

- Per-persona validation dispatches (`*-persona-0N-*`) — each persona's live-dispatch output against its target eval case, with verdict.
- Review prompts (`*-review-prompt.md`, `*-panel-final-review-prompt.md`) — the prompts pasted into separate Claude Code sessions for cross-cutting verification.
- Smoke-test cycles (`*-orchestrator-smoke-test-pr005*.md`) — the three end-to-end orchestrator dispatches against the PR #5 pre-fix code, showing precision/recall measurements and bug fixes across cycles.

## Caveats for readers

- **Absolute paths in some files may not resolve.** The review-prompt files contain instructions for the original session, including absolute paths like `D:/source/work/marketer-agents/...`. These reflect the author's local environment at the time of authoring; current canonical docs (`personas/`, `orchestrator/`, `HANDOFF.md`) use relative or placeholder paths.
- **Persona versions referenced here are point-in-time.** A file titled `persona-01-v0.1-...` documents persona #1 *as it was* at v0.1; the current persona file is a later version. Follow the persona file's changelog frontmatter to trace evolution.
- **Author/reviewer names** in some files (e.g., references to "the human reviewer") have been preserved with the framing of the time. The narrative focuses on the failure pattern and how it was caught, not on credit/attribution — see git history for actual authorship.

## Where to start for current state

If you want to understand the panel as it ships today, do NOT start here. Start with:

1. [`/README.md`](../README.md) — external entry point.
2. [`/HANDOFF.md`](../HANDOFF.md) — internal future-self handoff with current Phase 2.5 status.
3. [`/personas/*.md`](../personas/) — the live persona prompts.
4. [`/orchestrator/orchestrator-prompt.md`](../orchestrator/orchestrator-prompt.md) — the live runbook.

`eval-runs/` is the audit trail for *how* the panel got to its current state, not the panel's current state itself.
