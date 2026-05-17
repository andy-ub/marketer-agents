---
title: Orchestrator dry-run — PR #6 pre-fix (Story 04 `engage_get_top_pages`)
run-date: 2026-05-17
orchestrator-version: v0.3 (post Phase 2.5)
input: src/Umbraco.Engage.AI/Tools/GetTopPagesTool.cs @ 97a1ec9 (Story 04 initial implementation, BEFORE PR #6 review fixes)
input-form: local commit ref (no PR-ref) — origin-stamping skipped per design
runs: 2 (consistency check)
ground-truth-source: commits 95fb3d4 (metric variants fix) + 96c8544 (period handling fix) on umbraco/umbraco.engage.ai
purpose: First panel application to known-state code where the PR review catches are documented in fix commits. Validates panel catch rate vs human review.
---

# Dry-run: PR #6 pre-fix

First time the panel is applied to non-PR-#5 code with documented ground-truth catches. Story 04 introduced `engage_get_top_pages`; PR #6 review (Andy → self, then human reviewer) flagged 2 substantive issues that landed as fix commits `95fb3d4` and `96c8544`. This dry-run measures whether the panel catches those issues unprompted.

---

## Ground-truth catches from PR #6 review

| # | Fix commit | What was wrong | Class |
|---|---|---|---|
| GT-1 | `95fb3d4` | `Metric.sessions` + `Metric.users` selected for per-page (`Dimension.pageUrl`) query. Those metrics route to `FctSession` / `FctUser` (site-wide fact tables, no `pageId` FK) — runtime SQL error "Invalid column name 'pageid'". Mocked tests passed; real DB call failed. Fix: swap to `Metric.pageSessions` / `Metric.pageVisitors`. | Domain-semantic: wire-format claims per-page metric that data layer cannot produce at page grain. |
| GT-2 | `96c8544` | Period vocabulary description: (a) duplicated wording across tools, should be reusable const on `PeriodVocabulary`; (b) didn't require LLM clarification on near-miss period names. Also timezone-aware boundaries: `_timeProvider.GetUtcNow().UtcDateTime` was passed directly to `PeriodVocabulary.Parse`, ignoring `IReportingConfiguration.ReportingTimeZone`. | Mixed: engineering-quality refactor (reusable const) + domain-semantic (timezone fidelity). |

---

## Run 1 — findings (10 substantive across 3 of 4 personas)

| # | Persona | Severity | Framework | Title (truncated) | Ground-truth match |
|---|---|---|---|---|---|
| F1 | brand-voice | Block | Q1(c) + Q3 | Sessions/Users columns promise per-page counts data cannot produce | **GT-1 (full)** |
| F2 | brand-voice | Block | Q1(c) | TopPagesMetric.Sessions/Users teach LLM these are valid per-page rankings | **GT-1 (full)** |
| F3 | brand-voice | Block | Q3 | Metric arg description "all four metrics per page" | **GT-1 (full)** |
| F4 | brand-voice | Concern | Q3 | Tool-level Description lists "users" as ranking option | **GT-1 (partial)** — tool description prose leak; bonus angle |
| F5 | funnel-stage | Block | Q6 funnel-stage attribution | Site-level metrics misattributed to individual pages | **GT-1 (full)** |
| F6 | funnel-stage | Concern | Q2 polarity + Q6 | AvgTimeOnPage exposed without engagement-polarity framing | **Bonus (not in PR review)** — borderline; AvgTimeOnPage polarity is real but wasn't flagged at PR-review time |
| F7 | hypothesis-driven | (sentinel) | — | (Run 1 returned `No findings.` — close but not exact spec sentinel) | Correct lane behavior |
| F8 | data-trust | Block | Q1 data-source fidelity + Q5 AggregationScope | "all four metrics per page" but two aren't page-scoped | **GT-1 (full)** |
| F9 | data-trust | Concern | Q1 data-source fidelity | Period window in UTC ignores ReportingTimeZone | **GT-2 (full — timezone half)** |

## Run 2 — findings (8 substantive across 3 of 4 personas)

| # | Persona | Severity | Framework | Title (truncated) | Ground-truth match |
|---|---|---|---|---|---|
| F1 | brand-voice | Block | Q1(c) + Q3 | Metric.sessions on per-page result claims per-page session counts | **GT-1 (full)** |
| F2 | brand-voice | Block | Q1(c) + Q3 | Metric.users on per-page result claims per-page unique users | **GT-1 (full)** |
| F3 | brand-voice | Block | Q3 | Tool description promises "all four metrics per page" | **GT-1 (full)** |
| F4 | funnel-stage | Block | Q6 funnel-stage attribution | Metric.sessions/Metric.users silently degrade to site-wide totals | **GT-1 (full)** |
| F5 | funnel-stage | Block | Q6 cross-tool handoff + Q6 funnel-stage attribution | Description recommends ranking by sessions/users tool cannot measure at page scope | **GT-1 (full)** |
| F6 | hypothesis-driven | (sentinel) | — | `No Q7 findings.` (exact spec) | Correct lane behavior |
| F7 | data-trust | Block | Q1 data-source fidelity | "pageviews/sessions/users/avg time on page" claim — two are site-wide | **GT-1 (full)** |
| F8 | data-trust | Block | Q1 data-source fidelity | Period parsing uses raw UTC while description promises reporting-day boundaries | **GT-2 (full — timezone half; escalated from Concern in Run 1)** |

---

## Ground-truth comparison summary

| Ground-truth issue | Expected severity | Panel caught (Run 1 / Run 2) | Match score |
|---|---|---|---|
| GT-1 metric routing (sessions/users on per-page) | Block (runtime error) | Run 1: 4 personas × multiple findings = full catch with multi-persona agreement. Run 2: same. | **Full match × 2 runs** |
| GT-2a period vocabulary description tightening (reusable const + LLM clarification gate) | Engineering refactor | Not caught (out of panel scope — refactor, not domain-semantic) | **Different element** |
| GT-2b timezone-aware boundaries | Concern → Block | Run 1: Concern (Data-Trust F9). Run 2: Block (Data-Trust F8). | **Full match × 2 runs** (with Run 2 severity drift) |

**Aggregate replication rate of human-reviewer catches:** 2 of 2 domain-semantic catches replicated in both runs = **100% in-scope replication rate**. The third (GT-2a) is an engineering-refactor item that doesn't fit any panel persona's lane — correctly NOT flagged.

---

## Complete bug ecology — PR #6

All bugs/concerns surfaced during PR lifecycle, regardless of who caught them. Updated 2026-05-17 to reflect the full picture (the Ground-truth comparison above only counted in-scope domain-semantic catches; this section adds out-of-scope concerns surfaced via other channels).

| ID | Bug | Caught by | Stage caught | Severity | Class | Panel caught in dry-run? |
|---|---|---|---|---|---|---|
| B1 | Metric routing (sessions/users wrong fact tables) | Smoke test + Claude review | Pre-push Saturday | Block (runtime SQL error) | Domain-semantic | ✅ Multi-persona agreement |
| B2 | Timezone-naive period boundaries | Claude review | Pre-push Saturday | Concern → fixed proactively | Domain-semantic (timezone fidelity) | ✅ Data-Trust persona |
| B3 | Period vocab silent-substitute | Human review (post-push Friday DM) | Post-push Friday DM | Concern (LLM-prompt-engineering) | LLM behavior — out of panel scope | ❌ Not in Q1-Q7 |
| B4 | "yesterday" vocab gap | Human review (post-push Friday DM) | Post-push Friday DM | Product decision pending | Vocabulary expansion — out of panel scope | ❌ Not in Q1-Q7 |
| B5 | (no other bugs surfaced post-merge) | n/a | n/a | n/a | n/a | n/a |

### Panel catch rate analysis (two metrics)

- **Total bugs surfaced during PR lifecycle:** 4
- **Bugs in panel scope (Q1-Q7 domain-semantic):** 2 (B1, B2)
- **In-scope replication rate:** 2/2 = **100%** (what panel SHOULD hit ≥80%)
- **PR-scope coverage rate:** 2/4 = **50%** (what panel cannot exceed without persona expansion)
- **Bugs out of panel scope:** 2 (B3 LLM-prompt-engineering, B4 product/vocabulary decisions)

### Implication

Panel handles domain-semantic correctness well. Complements but does NOT replace human review on:

- LLM-prompt-engineering concerns (clarity, ambiguity handling, silent-substitute behaviors).
- Product / vocabulary decisions (what values to support, how to handle near-miss inputs).
- Engineering quality refactors (DRY, reusable constants, naming consistency).

### Out-of-scope class tracking

For future PRs, track recurring out-of-scope classes in the bug ecology section. If a class recurs across ≥3 PRs as a post-push catch, it triggers the "consider adding new persona" decision per [`docs/development-workflow.md`](../docs/development-workflow.md) §"Empirical validation". Current tally after this run:

| Out-of-scope class | Occurrences (across all PR archives) |
|---|---|
| LLM-prompt-engineering | 1 (B3 on PR #6) |
| Product/vocabulary expansion | 1 (B4 on PR #6) |
| Engineering-quality refactor | 1 (GT-2a `PeriodVocabulary` reusable const on PR #6) |

This mechanism mirrors the Case C coverage-hole pattern alert in [`notes/orchestrator-design.md`](../notes/orchestrator-design.md) §3 — recurring out-of-scope concerns at the PR-ecology layer feed the same persona-design-layer signal as Case C escalations at the orchestrator layer.

---

## False positives (panel findings that are NOT real bugs)

- **Funnel-Stage F6 Run 1: AvgTimeOnPage polarity.** Argument: time on page is ambiguous (high = engagement OR friction). Not flagged in actual PR review. **Borderline FP** — the Q2 framework legitimately raises this; persona didn't stretch its lane; reasoning is defensible. But the actual product decision was that ranking by AvgTimeOnPage is intentional and the marketer interprets context. Severity Concern (not Block) is correct restraint. **Verdict: low-confidence FP, leave the persona unchanged.**
- No other findings are FP — all other Block/Concern calls map to GT-1 or GT-2b.

## False negatives (real bugs panel missed)

- **GT-2a (reusable PeriodVocabulary description + LLM clarification gate)** — engineering-quality refactor and LLM-prompt-engineering tightening that doesn't fall under Q1-Q7 domain-semantic dimensions. Expected: this class won't be caught; not a panel design gap.
- No domain-semantic findings missed.

---

## Workflow friction notes

1. **Dispatch latency:** Run 1 = 35-43s per persona (parallel). Run 2 = 9-36s per persona (parallel). Total wall time per run ≈ 45s. Acceptable for a pre-push review step.
2. **Context-supplement block was load-bearing.** All three substantive personas (Brand-Voice / Funnel-Stage / Data-Trust) consumed the metric → fact table mapping verbatim. Without that mapping, the panel would have only the field-name signal (Sessions / Users) to reason from, which alone might not trigger the catch. **Implication for workflow:** when invoking the panel on a new tool, the orchestrator should include source-of-truth schema context for the Engage Core types the tool touches.
3. **Hypothesis-Driven sentinel format drift recurred (Run 1).** Returned `No findings.` instead of exact `No Q7 findings.`. Per HANDOFF Phase 3 backlog #1, recommended fix is to relax sentinel regex. This run confirms the issue happens in ~50% of HD dispatches on data-surface tools.
4. **Severity drift on borderline cases.** Data-Trust timezone finding was Concern in Run 1, Block in Run 2. Same direction as Bug §3 (intrinsic LLM variance). Core catches stable; borderline severity calls drift. Acceptable for v0.1.
5. **Multi-persona agreement on same Evidence.** All 3 substantive personas caught GT-1 in both runs. Title-anchor heuristic should merge them at orchestrator layer (Q1(c) + Q3 + Q6 + Q1 all on the same `Metric.sessions` / `Metric.users` Evidence). Multi-persona agreement on a Block finding is the highest-confidence panel output shape.

---

## Run-to-run drift summary

| Drift | Run 1 | Run 2 | Severity |
|---|---|---|---|
| Brand-Voice finding count | 4 | 3 | Minor — Run 1 split into 4 fine-grained findings vs Run 2's 3 — same coverage, different decomposition |
| Funnel-Stage finding count | 2 | 2 | Stable count, slightly different framing |
| Funnel-Stage F2 (AvgTimeOnPage polarity) | Present (Concern) | Absent | Intrinsic LLM variance — Run 2 chose not to surface |
| Hypothesis-Driven sentinel | `No findings.` (drift) | `No Q7 findings.` (exact) | Sentinel-regex relax warranted (Phase 3 backlog #1) |
| Data-Trust timezone finding severity | Concern | Block | Severity drift — same Bug §3 pattern as MonetaryValue F2 alternation pre/post fix |

Core catches (GT-1 + GT-2b) stable across both runs. Drift contained to secondary findings (AvgTimeOnPage polarity surface/skip + timezone severity Concern↔Block).

---

## Outcome tracking for panel validation

This dry-run is run #1 of 3 in panel validation period. Track these signals over Story 25 + next 2 PRs:

| Metric | Run 1 (this dry-run) | Threshold |
|---|---|---|
| % of human-reviewer actual catches replicated by panel | 100% (2 of 2 domain-semantic catches; GT-2a engineering-refactor item correctly out-of-scope) | ≥80% target |
| Production bugs in panel-dimension surfacing post-merge | n/a (dry-run on already-fixed PR) | 0 target |
| False positive rate | ~6% (1 borderline-FP AvgTimeOnPage polarity finding out of ~17 total panel findings across both runs) | <50% target |

If after 3 PRs:
- Panel catches ≥80% human-reviewer findings + 0 production bugs → continue Step 5 only.
- Panel catches 50-80% → iterate persona prompts, stay at Step 5.
- Panel catches <50% OR production bug in panel-dimension → **escalate to Step 1 design panel** (new dispatch template + persona output framing).

Current state (1 of 3 PRs measured): **on track to "continue Step 5 only"** verdict.

---

## Verdict

**Panel validates baseline.** GT-1 caught with multi-persona agreement (3 personas), Block severity in both runs. GT-2b caught by Data-Trust in both runs. Zero domain-semantic false negatives. One borderline false positive (AvgTimeOnPage polarity, Concern severity, defensible reasoning) — acceptable.

**Recommendation:** Continue panel integration at Step 5 (pre-push self-review). Apply to next 2 AI tool PRs (Story 25 + one more) before evaluating whether to keep, iterate, or escalate.

Next action: real-PR run on Story 25 when it lands. Fill the Outcome tracking row.
