---
title: Synthetic Q7 eval — recommendation without hypothesis-echo on GetSegmentsTool
id: synthetic-q7-segments-tool
case-type: failure-prevention (synthetic — no real PR source)
audit-question: Q7 — Hypothesis echo at the recommendation layer
source-pr: N/A (synthetic diff constructed for panel persona #3 validation)
source-fix: N/A
reviewer-context: Persona #3 (Hypothesis-Driven Marketer) validation; no PR #5 eval covers Q7
created: 2026-05-16
purpose: Persistent eval input for the Hypothesis-Driven Marketer persona. Engage currently has no production tool that authorizes recommendations without hypothesis-echo shape, so Q7 validation requires a synthetic diff. This file lifts the synthetic GetSegmentsTool used in `eval-runs/2026-05-16-persona-03-v0.1-validation.md` and the v0.2 re-dispatch into a persistent eval input.
---

# Eval case: recommendation authorized without hypothesis-echo (synthetic GetSegmentsTool)

## What is the synthetic code

A hypothetical `GetSegmentsTool.cs` constructed to stack three independent Q7 violations. Not a real Engage tool — Engage's current segment surface does not authorize recommendations in this shape.

```csharp
namespace Umbraco.Engage.AI.Tools;

[AITool("engage_get_segments", "Get Engage Segments", ScopeId = EngageReadScope.ScopeId)]
public class GetSegmentsTool : AIToolBase<GetSegmentsArgs>
{
    private readonly ISegmentService _segments;
    private readonly IGoalService _goals;

    public GetSegmentsTool(ISegmentService segments, IGoalService goals)
    {
        _segments = segments;
        _goals = goals;
    }

    public override string Description =>
        "Use this when the user asks about audience segments and how they are performing relative to goal targets. " +
        "Returns each Engage segment with its name, size (visitor count over the reporting window), conversion rate, " +
        "and a suggested action describing what the marketer should do next. " +
        "For segments where the conversion rate is below the configured goal target, recommend the marketer test " +
        "alternative messaging or pause spend to that segment while investigation completes. " +
        "Always rank segments by absolute size and highlight the largest as the priority for action.";

    public override async Task<IResult> ExecuteAsync(GetSegmentsArgs args, CancellationToken ct)
    {
        var segments = await _segments.GetAllAsync(ct);
        var target = await _goals.GetConfiguredTargetAsync(args.GoalKey, ct);

        var results = segments
            .Select(s => new SegmentResult(
                Key: s.Key,
                Name: s.Name,
                Size: s.VisitorCount,
                ConversionRate: s.ConversionRate,
                SuggestedAction: s.ShouldInvestigate
                    ? "Investigate and test alternatives"
                    : "Maintain current spend"))
            .OrderByDescending(r => r.Size)
            .ToList();

        return Results.Ok(results);
    }
}

public record SegmentResult(
    Guid Key,
    string Name,
    int Size,
    decimal ConversionRate,
    string SuggestedAction);
```

## The three stacked Q7 violations

1. **Description authorizes a bare recommendation.** *"For segments where the conversion rate is below the configured goal target, recommend the marketer test alternative messaging or pause spend to that segment while investigation completes."* — names an action (test messaging) and an alternative action (pause spend) with no predicted lift, no baseline reference, no test method, no confidence tier.
2. **`SuggestedAction` is free-form prose.** `string SuggestedAction` on `SegmentResult` accepts any prose. Two invocations against the same segment can return structurally different recommendations because the schema does not enforce hypothesis shape (no `PredictedOutcome` / `Baseline` / `TestMethod` / `ConfidenceTier` fields).
3. **"Always rank by size, highlight the largest as priority."** Forces a recommendation independent of sample size, conversion-rate confidence, or goal-distance materiality. Removes every threshold gate the marketer needs to read the prioritization as actionable vs noise.

## Why it fails

Q7 — hypothesis echo at the recommendation layer — requires that LLM-facing tools producing or authorizing actionable recommendations enforce **hypothesis-echo shape** on the recommendation output. Two acceptable structurally-equivalent formats per audit rule v1.1:

- **IF/THEN/BECAUSE form** — `IF [action], THEN [outcome], BECAUSE [evidence]` + confidence tag + test method.
- **"By doing X, we believe Y will happen. If we are right, we expect Z."** — where `Z` is a quantitative expectation against named baseline.

A recommendation that names an action without naming the predicted outcome, the comparison baseline, or the validation method is not a recommendation a marketer can falsify. Resources committed against the action (creative brief, budget pause, sprint allocation) produce results the marketer cannot read as success or failure because there was no predicted result to compare against.

This synthetic stacks all three failure modes:
- Prose-level violation (description authorizes vague recommendation).
- Schema-level violation (free-form string field cannot carry shape).
- Threshold-gate violation (forced recommendation regardless of evidence sufficiency).

## What a marketer should ask (failure scenario without fix)

> Marketer: "Which segment should we focus on this quarter?"
>
> Tool (as written above): returns segments ranked by size; the largest segment has `SuggestedAction = "Investigate and test alternatives"`.
>
> LLM interprets: "Your largest segment, [Name], is the priority for action — investigate and test alternatives."
>
> Reality: the marketer commissions a discovery round (analyst hours, test plan, creative brief, possibly a media-buying pause). After two weeks, conversion rate has moved by some amount. The marketer cannot answer:
>
> - What lift was expected?
> - Against which baseline period?
> - Was the move bigger or smaller than predicted?
> - Did the new messaging cause the move, or was the segment regressing/recovering on its own?
>
> Either result — numbers up or numbers down — is ambiguous. The next decision on the same segment starts from the same blind spot. No accumulating learning across invocations.

The wasted action: every recommendation invocation costs the marketer real resources (analyst time, creative spend, budget reallocation) against unfalsifiable suggestions. Over time, trust in the tool's recommendations erodes — not from a single dramatic miss but from accumulated ambiguity.

## Audit question (v1.1)

**Q7 — Hypothesis echo at the recommendation layer.** Does the tool surface, when producing or authorizing actionable recommendations, enforce hypothesis-echo shape (predicted outcome + baseline + test method + confidence) on the output?

For this synthetic tool: **NO at three independent layers** (prose description, schema, forced-recommendation gate).

Note: Q7 has the strongest cross-source agreement in the marketer-agent-research survey (7/9 OSS sources). Hypothesis-echo discipline is the single most widely-validated marketer-agent pattern. Q7 violations have the highest cross-citation weight in the audit framework.

## Expected reviewer output

A Hypothesis-Driven Marketer panel persona reviewing this synthetic diff should produce three findings, each citing Q7:

```
FINDING 1 — Description authorizes recommendations without hypothesis-echo shape
  Severity: Block
  Cites: Q7 hypothesis echo
  Catches: the "recommend... test alternative messaging or pause spend" prose
  Recommends: replace with hypothesis-echo template (IF/THEN/BECAUSE or "By X we expect Y")
  + baseline + test method + confidence tier

FINDING 2 — SuggestedAction is free-form prose with no shape constraint
  Severity: Block
  Cites: Q7 hypothesis echo
  Catches: stringly-typed SuggestedAction on SegmentResult
  Recommends: split into shape-bearing fields — PredictedOutcome, Baseline, TestMethod, ConfidenceTier

FINDING 3 — "Always rank by size, highlight largest as priority" forces recommendation
  Severity: Block
  Cites: Q7 hypothesis echo (below-threshold gate)
  Catches: "Always... priority for action" instruction
  Recommends: gated prioritization with sample-size / variance / window thresholds;
  inconclusive label when gates fail
```

All three Block calls pass the rubric's standalone test: each independently commits the marketer to irreversible resources (creative brief, budget pause, sprint allocation) against a recommendation that cannot be falsified.

## Mechanical eval criterion

When this eval case is fed to the Hypothesis-Driven Marketer persona:

**PASS:** the persona produces at least Findings 1 and 2 above (description-level + schema-level violations). Finding 3 (below-threshold gate) is a bonus catch — it is in the framework's scope but the eval is satisfied without it. Each finding must cite Q7 verbatim and recommend hypothesis-echo shape per the framework's two acceptable formats. Voice must follow consequence-first template (Sentence 1 leads with the marketer's resource commitment against an unfalsifiable recommendation, not with the schema description).

**FAIL:** the persona misses the recommendation-without-shape failure mode entirely, OR produces findings outside Q7's lane (Q1 / Q2 / Q3 issues — there are none in this diff by construction), OR produces findings without Q7 framework citation, OR uses flat schema-paraphrase voice instead of consequence-first.

## Validation history

| Date | Persona version | Result | Notes |
|---|---|---|---|
| 2026-05-16 | 03-hypothesis-driven-marketer v0.1 | PASS | Initial positive validation. Synthetic diff matched Worked Example A's structural shape; anchor-leak suspected. See `eval-runs/2026-05-16-persona-03-v0.1-validation.md`. |
| 2026-05-16 | 03-hypothesis-driven-marketer v0.2 | PASS | Re-dispatch after stripping the fix-portion of Worked Example A. Finding 1 Recommendation produced "By [action], we expect [outcome]" cadence (the second framework format), not the IF/THEN/BECAUSE format the stripped Worked Example A used — confirms framework prose alone drives the Recommendation, not example-anchor echo. See `eval-runs/2026-05-16-persona-03-v0.2-redispatch.md`. |

## Related

- `feedback_domain_field_audit.md` v1.1 Q7 — canonical Q7 definition with both hypothesis-echo formats.
- `marketer-agent-research/notes/marketer-agent-survey.md` — Q7 OSS convergence evidence (7/9 sources).
- `personas/03-hypothesis-driven-marketer.md` — the persona this eval validates.
- `notes/orchestrator-design-notes.md` §5 — Q5 forward-looking trigger criterion (analogous pattern for Q5 deferred eval).
