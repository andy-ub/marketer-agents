# Marketer-Panel Orchestrator

The orchestrator dispatches the 4-persona panel in parallel, merges findings, stamps origin metadata, and emits a markdown panel review.

## Files

- [`orchestrator-prompt.md`](orchestrator-prompt.md) — the main runbook. Claude reads this and executes each step in order.
- [`persona-dispatch-template.md`](persona-dispatch-template.md) — boilerplate prompt sent to each persona subagent during dispatch.
- [`regex-patterns.md`](regex-patterns.md) — parsing reference for finding template, sentinels, context-gap lines, evidence tokens.
- [`origin-stamping-subsystem.md`](origin-stamping-subsystem.md) — Step 5 spec: origin assignment logic, failure path, gh CLI dependency, worked example.

State files (created on first orchestrator run):
- `../notes/coverage-hole-log.md` — append-only audit trail of Case C escalations.
- (optional) `../eval-runs/<date>-orchestrator-<pr-ref>.md` — archive of rendered panel reviews.

## How to invoke

In a Claude Code session opened at this repo's root, ask Claude:

> "Run the marketer panel on `umbraco/umbraco.engage.ai#5`."

Or use the local-form input:

> "Run the marketer panel on the diff in `eval-inputs/pr-005-value-monetaryvalue.md` (pre-fix snippet)."

Or paste a diff directly into chat:

> "Run the marketer panel on this diff:
> ```csharp
> <paste>
> ```"

Claude follows `orchestrator-prompt.md` Step 0 (pre-flight checks) through Step 7 (render).

## Dependencies

- **`gh` CLI (optional):** required for origin-stamping. If absent, all findings get `origin: unknown` and `run_health.origin_stamping = skipped (gh unavailable)`. Panel still produces findings, just without origin metadata. Install via `winget install GitHub.cli` if origin-stamping value matters for your reviews.
- **`git`:** required for local-form input (commit-ref + path).
- **`Agent` tool:** the orchestrator dispatches 4 personas via `Agent` calls. Inherits the parent session's tool availability.

## Output

Markdown panel review printed in-session. Structure: Summary / Findings (sorted by severity + multi-persona agreement) / Context-gap routing / Run health. See `orchestrator-prompt.md` Step 7 for the exact template.

## Smoke-test status

Phase 2 implementation complete (2026-05-16). Phase 3 smoke test (against PR #5 pre-fix code) pending. Smoke-test attention points are documented at the bottom of `orchestrator-prompt.md`.
