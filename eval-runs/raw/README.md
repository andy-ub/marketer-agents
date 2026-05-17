---
title: Raw subagent outputs
purpose: Verbatim subagent text outputs preserved alongside builder summary archives so independent reviewers can audit reasoning-quality claims against the actual artifact, not the builder's classification of it.
created: 2026-05-17
trigger: Session 3 transparency review (v0.5 sign-off) flagged that builder summary archives are unfalsifiable when raw text is summarized away. Future reviews need source artifacts.
---

# Raw subagent outputs

This directory holds verbatim, unedited responses from each persona subagent dispatch. Files are written by the orchestrator at Step 2 (dispatch_personas_in_parallel) — see [`../../orchestrator/orchestrator-prompt.md`](../../orchestrator/orchestrator-prompt.md) §"Raw-output preservation".

## File naming

```
<YYYY-MM-DD>-<input-id>-<persona-id>-run<N>.md
```

- `<input-id>` — `pr<NUMBER>` for PR-ref form, short commit SHA for local-commit form, `local-diff-<HHMM>` for pasted form.
- `<persona-id>` — one of `brand-voice`, `funnel-stage`, `hypothesis-driven`, `data-trust`.
- `<N>` — run-within-dispatch-session counter (1 for single run; 1 and 2 for two-run consistency checks).

## File contents

Each file contains the verbatim subagent response text, prepended with a minimal YAML header (`persona`, `run-date`, `dispatch-status`). No analysis, no classification — just the artifact.

Failed dispatches (timeout, malformed) are preserved identically with `dispatch-status` set to `timeout` or `malformed`. Failure artifacts are also evidence — they show what the subagent did or did not produce when it was supposed to fire.

## Why this matters

Builder summary archives in `../*.md` classify each subagent's output (Reasoning "Genuine" / "Post-hoc" / "Hybrid", Considered-but-not-flagged "useful audit signal" / "out-of-lane filler"). Those classifications are useful — they're how the builder communicates verdicts forward. But classifications are not artifacts. An independent reviewer auditing a transparency claim ("12 of 14 Reasoning fields were genuine") needs to read the 14 Reasoning bodies themselves, not the builder's tally.

Without raw preservation, every transparency-quality claim becomes:
- Falsifiable only by re-dispatch (non-deterministic — LLM sampling drifts across runs)
- Or auditable only via the builder's own classification (auditing the auditor, not the artifact)

This preservation pattern fixes that gap.

## Retroactive availability

Preservation began **2026-05-17** following Session 3 transparency review. Runs prior to that date — including the 2 PR #6 transparency runs that motivated this change — do **NOT** have raw artifacts. Their summary archives (`../2026-05-17-orchestrator-pr006-rerun-with-reasoning.md`) are the only record. Treat any pre-2026-05-17 reasoning-quality claim as builder-classified, not independently auditable.

## Workflow

The orchestrator writes these files automatically — you do not produce them by hand. If you find yourself manually authoring a raw file, the orchestrator workflow has been bypassed and the artifact's provenance is suspect.

When a raw file is missing for a summary archive that should have one: the orchestrator either skipped the preservation step (bug) or the dispatch happened before preservation existed (pre-2026-05-17). Both are tractable; both should be surfaced rather than papered over.
