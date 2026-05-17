---
title: Multi-persona merge bloat stress test — GT-1 (3-persona Block on metric routing)
run-date: 2026-05-17
trigger: Session 3 transparency review raised concern that "when 3 personas flag the same Evidence with Reasoning fields, the orchestrator's merge stacks 3 × Reasoning bodies on the single rendered Block finding."
test-mode: synthetic — raw Reasoning bodies from PR #6 transparency runs are not preserved on disk (pre-dated the raw-artifact preservation pattern added 2026-05-17). Synthetic Reasoning bodies constructed from the builder summary archive's descriptive classifications + template-typical word counts (~90 words per Reasoning field from the +38% bloat measurement).
---

# Multi-persona merge bloat — stress test on GT-1

## Setup

GT-1 (PR #6 metric routing — `Metric.sessions` / `Metric.users` on per-page query) was caught by 3 personas in both transparency-run dispatches: Brand-Voice (Q1(c) + Q3), Funnel-Stage (Q6 funnel-stage attribution), Data-Trust (Q1 data-source fidelity + Q5 AggregationScope). This is the canonical multi-persona-agreement Block — the highest-confidence panel output shape.

If multi-persona Reasoning stacking is a bloat problem, GT-1 is where it would show. Test the merge step rendering on that finding.

## Stress test — synthetic Reasoning bodies

The raw subagent outputs for PR #6 transparency runs were NOT preserved on disk (the preservation pattern was added in the same session that motivated this test — see [`raw/README.md`](raw/README.md) §"Retroactive availability"). So this test runs synthetic Reasoning bodies constructed from the builder summary archive's descriptive classifications. Each synthetic Reasoning is calibrated to template-typical word count (~90 words, derived from the +38% finding-level bloat measurement minus the existing Issue + Recommendation + structured-metadata budget).

### Synthetic Brand-Voice Reasoning (Q1(c) + Q3)

> Framework trigger: `TopPageResult.Sessions` and `TopPageResult.Users` are field names on a per-page record (page dimension keyed by `pageUrl`). Q1(c) and Q3 both fire: the field names claim a Stone-class per-page count, but the underlying fact tables (`FctSession`, `FctUser`) have no `pageId` FK and resolve at site grain only. Alternative considered: could the persona read these as legitimately ambiguous (LLM might infer site-scope from context)? Rejected — the column lives on a record whose other rows are page-scoped, and the LLM will narrate row-as-a-whole at the record's nominal grain. Severity boundary: Block, not Concern — the marketer will receive a per-page sessions narration the data cannot back, and the runtime SQL error makes the failure detectable but only post-deployment.

(~95 words.)

### Synthetic Funnel-Stage Reasoning (Q6)

> Framework trigger: the tool's stated scope is per-page ranking (`Dimension.pageUrl`), but `Metric.sessions` / `Metric.users` route to site-wide fact tables. Q6 funnel-stage attribution fires: a metric reading at one stage (page) cannot be backed by data at a different stage (site). Alternative considered: is this a Q2 polarity issue (high count good/bad)? Rejected — polarity is unaffected; what's broken is the *attribution scope* of the count, not its direction. Severity boundary: Block, not Concern — the LLM will rank pages by metrics that silently aggregate site-wide, the marketer will reallocate against rankings that have no per-page resolution.

(~95 words.)

### Synthetic Data-Trust Reasoning (Q1 data-source fidelity + Q5 AggregationScope)

> Framework trigger: tool description promises "all four metrics per page" but the data layer for `Metric.sessions` / `Metric.users` cannot produce at page grain. Q1 data-source fidelity fires (description overclaims source), Q5 AggregationScope fires (the metric's actual aggregation scope is `Blended` site-wide, not `Page`). Alternative considered: is this an engineering bug rather than a data-trust framing issue? Rejected — Q1's lane is wire-format claims that the data layer does not back; the description IS the wire format, and the overclaim IS the lane. Severity boundary: Block, not Concern — marketer commits resources against a "per-page ranking" that the SQL layer will fail to produce or will silently aggregate.

(~110 words.)

## Render under current orchestrator spec (Step 4b + Step 7)

Re-reading the current spec carefully:

- **Step 4b** (compound severity): *"any Block in group: Block; fold other findings' Recommendations into the Block finding via 'Related concerns from {persona-ids}: …'"* — folds **Recommendations**, not Reasonings.
- **Step 7 render template**: shows the lead persona's Reasoning + Issue + Recommendation. Related concerns appear only as `*Related concerns from <other-persona>: <other-persona's Recommendation>*` — Recommendation only.

So under the current spec, the merged GT-1 Block finding contains **one Reasoning body** (the lead persona's, presumably Brand-Voice as panel-slot 1 with multi-persona agreement). Non-lead Reasonings are dropped from the rendered output.

### Word count under current rule

| Component | Words (synthetic estimate) |
|---|---|
| Structured metadata (Severity, Evidence, Framework citation, Origin, Compound flags, Multi-persona agreement, Persona severity) | ~60 |
| Reasoning (lead persona only) | ~95 |
| Issue (lead persona) | ~120 |
| Recommendation (lead persona) | ~110 |
| Related concerns from FS (Recommendation only) | ~110 |
| Related concerns from DT (Recommendation only) | ~110 |
| **Total per merged Block finding** | **~605 words** |

A 600-word merged Block finding is at the edge of readable in scan mode but tractable when the marketer is actually deciding whether to merge. Not a wall of text.

### Information loss under current rule

The current rule **drops Funnel-Stage and Data-Trust Reasonings entirely**. This is the more interesting bloat-vs-signal trade-off Session 3 didn't surface. The lead persona's Reasoning explains *one* angle of why the catch fires; the dropped Reasonings explain Q6 and Q1 + Q5 angles that the lead's Q1(c) + Q3 framing does not cover.

For multi-persona-agreement Blocks specifically — the panel's highest-confidence output shape — dropping 2/3 of the framework-citation diversity in the Reasoning surface defeats the purpose of multi-persona agreement. The reader sees "3 personas flagged this" but only one persona's reasoning for why. Audit weakens.

## Hypothetical alternative — full Reasoning stacking

What if the merge stacked all 3 Reasonings (Session 3's worst-case assumption)?

| Component | Words |
|---|---|
| Structured metadata | ~60 |
| Reasoning (BV) | ~95 |
| Reasoning (FS) | ~95 |
| Reasoning (DT) | ~110 |
| Issue (lead) | ~120 |
| Recommendation (lead) | ~110 |
| Related concerns from FS | ~110 |
| Related concerns from DT | ~110 |
| **Total** | **~810 words** |

That IS approaching wall-of-text. A merged finding nearing 1,000 words for one Block when the run has ~10 findings = ~10K words of panel review. Scan-mode reading degrades; the marketer reaches for the summary table instead of the findings.

## Proposed merge rule (replaces Step 4b "Recommendations-only" fold)

Two changes:

### Rule A — keep all distinct framework citations across merged Reasonings (compression-preserving)

When folding a merge group, the merged Block finding renders:
1. Lead persona's full Reasoning (~95 words).
2. A `*Related reasoning from <other-persona> (<framework-citation>): <Reasoning compressed to 1-2 sentences naming the distinct framework angle>*` line per non-lead persona.

The compression: keep the framework citation + the alternative-considered + severity-boundary sentences; drop the framework-trigger sentence if it restates what the lead's Reasoning already established. Aim for ~30-40 words per non-lead Reasoning instead of the full ~95.

### Rule B — Recommendations stay as-is (no change to current spec)

Recommendation folding under the existing pattern remains.

### Estimated word count under Rule A

| Component | Words |
|---|---|
| Structured metadata | ~60 |
| Reasoning (lead, full) | ~95 |
| Related reasoning from FS (compressed) | ~35 |
| Related reasoning from DT (compressed) | ~35 |
| Issue (lead) | ~120 |
| Recommendation (lead) | ~110 |
| Related concerns from FS | ~110 |
| Related concerns from DT | ~110 |
| **Total** | **~675 words** |

11% bloat over current rule, but preserves the multi-persona-agreement audit signal that current rule discards. Acceptable trade-off.

## Recommendation

**Apply Rule A to orchestrator Step 4b + Step 7.** Keep all framework-citation diversity across merged Reasonings via short "Related reasoning from X" lines. Drop the framework-trigger sentence in compressed Reasonings if it duplicates the lead's angle.

This addresses Session 3's bloat concern (no 3 × full Reasoning stacking — never was per current spec) AND fixes the audit-signal loss Session 3 didn't surface (current rule drops non-lead Reasonings entirely, defeating multi-persona agreement's purpose).

## Caveats

1. **Synthetic, not measured.** Real PR #6 Reasoning bodies were not preserved. The word counts above are template-typical estimates calibrated to the +38% per-finding bloat the builder measured. Once the raw-artifact preservation pattern is live, re-run this test against real Reasoning bodies on the next multi-persona-agreement Block (likely Story 25 if its catches converge).
2. **Compression heuristic is unproven.** The "drop the framework-trigger sentence if duplicative" rule is an LLM-inferred heuristic, not a measured operation. The orchestrator may compress poorly. Watch the first 2-3 real applications.
3. **Rule applies only to multi-persona-agreement merges.** Single-persona findings render as-is. Two-persona merges render with one related-reasoning line. The bloat concern grows with group size; this rule's signal-preservation also grows with group size.

## Verdict

**Apply Rule A to orchestrator before Story 25.** Current rule under-preserves audit signal on the panel's highest-confidence output shape (multi-persona-agreement Block). Rule A adds ~70 words per merged Block but keeps the framework-angle diversity that multi-persona agreement is supposed to surface.

If Andy disagrees with the rule change and wants to keep the existing Recommendations-only fold for v0.5: that's defensible. Document the trade-off (current rule = simpler, drops non-lead Reasoning; Rule A = preserves diversity, 11% bloat). Don't apply Rule A silently — surface as a v0.5+ decision.

**Decision (Andy, 2026-05-17): Apply Rule A.** Rule A active in orchestrator-prompt.md from this commit forward. Re-test against real (not synthetic) Reasoning bodies on the next multi-persona-agreement Block in real-PR runs (likely Story 25 if its catches converge).
