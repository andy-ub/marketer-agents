---
title: Orchestrator smoke test — PR #5 pre-fix (Phase 2.5 complete)
run-date: 2026-05-17
orchestrator-version: v0.3 (post Bug §1 + §2 + §3 fixes)
input: src/Umbraco.Engage.AI/Tools/GetGoalsTool.cs @ d483e05^ (pre-fix code) — same as prior smoke tests
runs: 2 (consistency check)
persona-versions: brand-voice v0.4, funnel-stage v0.2, hypothesis-driven v0.2, data-trust v0.2
predecessors:
  - eval-runs/2026-05-17-orchestrator-smoke-test-pr005.md (pre Bug §1 fix)
  - eval-runs/2026-05-17-orchestrator-smoke-test-pr005-post-fix.md (post Bug §1 only)
---

# Phase 2.5 complete — final smoke test

Final smoke-test cycle. Validates Bug §2 (Data-Trust Q2 exclusion) + Bug §3 (Brand-Voice description-prose Block subcase) fixes; confirms Bug §1 (title-anchor heuristic) regression-free.

## Run 1 — findings (6 substantive + 1 sentinel)

| # | Persona | Severity | Framework | Notes |
|---|---|---|---|---|
| F1 | brand-voice | **Block** | Q3 + Q1(c) | MonetaryValue rename |
| F2 | brand-voice | **Block** ✅ | Q3 + Q1(c) | Description prose leak — Bug §3 verified |
| F3 | funnel-stage | Block | Q2 | IsInverted missing |
| F4 | funnel-stage | Concern | Q6 cross-tool handoff | Dangling pointer ("a separate tool covers...") |
| F5 | hypothesis-driven | (sentinel) | — | "No Q7 findings." exact |
| F6 | data-trust | Block | Q1 filter policy | IsInvalid missing + copy-paste anti-pattern cited |
| F7 | data-trust | Concern | Q1 data-source fidelity | Description silent on config-vs-tracking distinction |

Data-Trust Context gaps:
> `Context gap (out-of-lane): IGoal.IsInverted exists on the source-of-truth interface and GoalResult does not surface it — this is Q2 polarity territory and belongs to Funnel-Stage, not Q1. Routing rather than writing a finding, per the explicit Q2 exclusion in my lane definition.`

**Bug §2 verification:** ✅ Data-Trust correctly routed Q2 via Context gap (Case C) AND explicitly cited the new exclusion in its routing reason. Previous runs (pre-fix Run 2 + post-fix Run 2) had Data-Trust write a Q2 finding under "Q1 validity-adjacent" framing — now eliminated.

## Run 2 — findings (6 substantive + 1 sentinel)

| # | Persona | Severity | Framework | Notes |
|---|---|---|---|---|
| F1 | brand-voice | Block | Q3 + Q1(c) | MonetaryValue rename |
| F2 | brand-voice | **Block** ✅ | Q3 + Q1(c) | Description prose leak — Bug §3 verified Run 2 |
| F3 | funnel-stage | Block | Q2 | IsInverted polarity flag dropped |
| F4 | funnel-stage | Concern | Q6 cross-tool handoff | Named handoffs missing |
| F5 | hypothesis-driven | (sentinel) | — | **Minor:** "No findings — data-surface description; Q7 hypothesis echo not applicable." — close but not the exact spec'd sentinel |
| F6 | data-trust | Block | Q1 filter policy | IsInvalid missing |
| F7 | data-trust | Concern | Q1 data-source fidelity | `IsActive` reflects intent, not firing |

Data-Trust Context gaps:
> `Context gap (out-of-lane): IGoal.IsInverted is present on the domain entity and not surfaced on GoalResult. Polarity is Q2 territory — route to Funnel-Stage persona.`

**Bug §2 verification Run 2:** ✅ Same correct routing behavior. Two-run consistency confirmed.

## Cross-run consistency table

| Finding | Run 1 severity | Run 2 severity | Drift |
|---|---|---|---|
| BV-F1 MonetaryValue rename | Block | Block | — |
| BV-F2 Description prose | **Block** | **Block** | **✅ NO DRIFT** (Bug §3 fix consolidates this — previously alternated) |
| FS-F1 IsInverted | Block | Block | — |
| FS-F2 Handoff | Concern | Concern | — |
| HD sentinel | exact | minor format drift | new minor observation |
| DT-F1 IsInvalid | Block | Block | — |
| DT-F2 Data-source fidelity | Concern | Concern | — |
| DT Context gap routing Q2 | present | present | — |

**Severity drift on identical input:** 0 of 7 findings drift (was 3/7 pre-fix). Bug §3 fix is the decisive change — BV-F2 alternation eliminated.

## Bug §1 regression check (title-anchor heuristic)

Apply title-anchor heuristic to Run 1 findings (15 pairs from 6 substantive findings):

| Pair | Anchor overlap | Verdict | Truth | Match |
|---|---|---|---|---|
| BV-F1 ↔ BV-F2 | {monetary, value, currency} | RELATED | related (intended Monetary merge) | ✓ |
| All other 14 pairs | ∅ | NOT RELATED | independent | ✓ |

**Run 1 merge precision: 100% (1/1 correct), recall 100%.**

Run 2 follows the same pattern — only BV-F1↔BV-F2 merges, all others independent. **Run 2 precision: 100%, recall: 100%.**

Bug §1 fix holds across this third smoke-test cycle. No regression.

## Minor observations (Phase 3 backlog candidates, NOT blockers)

### Hypothesis-Driven sentinel format drift (Run 2 only)

Run 1 returned: `No Q7 findings.` (exact, matches spec)
Run 2 returned: `No findings — data-surface description; Q7 hypothesis echo not applicable.`

The persona prompt §"Sentinel for no-findings" specifies: *"respond with exactly: `No Q7 findings.` And nothing else."* Run 2 deviated. Intent equivalent; literal regex match fails.

**Implication for orchestrator parser:** Per [`orchestrator/regex-patterns.md`](../orchestrator/regex-patterns.md) §2 strict sentinel rule, Run 2 HD output would be flagged malformed → retry. Functionally a false-positive malformed flag.

**Mitigation options (Phase 3 backlog):**
- (A) Relax sentinel regex to match: `^\s*(No (Q[0-9c() ]+|in-scope|<dimension>) findings?\.?.*)\s*$` (allows trailing prose clarification).
- (B) Tighten persona prompt with stronger "do NOT embellish the sentinel" instruction.
- (C) Accept retry cost — one extra dispatch (~30s) per occurrence, low practical impact.

Recommend (A) — relaxed regex is cheapest and matches a clear semantic class without enabling other malformed shapes.

### Data-Trust no longer produces MonetaryValue Q1-data-source-fidelity finding

Phase 2.5 Run 1 + Run 2 Data-Trust both produced 2 findings (IsInvalid + data-source-fidelity-on-description). The pre-fix + post-fix runs had Data-Trust producing 3 findings, the third being "MonetaryValue overclaims the semantic of a free-form decimal" under Q1 data-source fidelity. This was a borderline Q3 (Brand-Voice's lane) reading dressed in Q1 framing.

Its absence in Phase 2.5 runs is not a fix from Bug §2 (that was for Q2). It may be:
- Sampling variance (Data-Trust just didn't surface that particular angle this time).
- Tighter Q1 vs Q3 mental boundary as a side effect of the Bug §2 exclusion making Data-Trust more careful about lane-stretching.

Either way it's a cleaner result. Worth monitoring on real PRs.

## Final Phase 2.5 status

| # | Bug | Status |
|---|---|---|
| 1 | Token-overlap heuristic over-merge | ✅ **FIXED** — title-anchor heuristic, 100% precision verified across 3 smoke-test cycles |
| 2 | Data-Trust Q1 lane stretching into Q2 | ✅ **FIXED** — explicit Q2 exclusion + Context gap routing verified across 2 Phase 2.5 runs |
| 3 | Brand-Voice F2 severity alternation | ✅ **FIXED** — description-prose Block subcase pinned, 0 alternation across 2 Phase 2.5 runs (was 100% alternation pre-fix) |

Phase 2.5 backlog **fully closed**. Three minor Phase 3 backlog items surfaced (sentinel regex relax + DT-F3 absence monitoring + nothing else).

## Verdict

**Panel + orchestrator ready for unattended use Monday. No open Phase 2.5 items. Real-PR validation is the next step.**
