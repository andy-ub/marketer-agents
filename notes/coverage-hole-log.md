---
title: Coverage-hole log
purpose: Append-only audit trail of Case C out-of-lane escalations pointing at dimensions no panel persona owns. Used by orchestrator/orchestrator-prompt.md Step 6c (check_coverage_hole_threshold).
format: one row per Case C escalation — `<run-date YYYY-MM-DD> | <pr-ref or "local-diff"> | <source-persona> | <target-dimension> | <evidence excerpt — first 100 chars>`
threshold-trigger: 3+ rows for the same target-dimension within the last 10 runs (or last 14 days, whichever window is shorter)
---

# Coverage-hole log

This file is automatically appended to by the orchestrator. The orchestrator-prompt.md Step 6 routes each `Context gap (out-of-lane):` escalation pointing at a no-owner dimension into a row below.

No rows yet — first orchestrator run will populate this file.

| Run date | PR ref | Source persona | Target dimension | Evidence excerpt |
|---|---|---|---|---|
