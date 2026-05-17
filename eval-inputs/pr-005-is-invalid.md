---
title: PR #5 eval — IsInvalid missing on Goal result
id: pr-005-is-invalid
case-type: failure-prevention
audit-question: Q1 — Repository filter policy
source-pr: umbraco/umbraco.engage.ai PR #5 (Story 03 — engage_get_goals)
source-fix: commit d483e05 (added IsInvalid + IsInverted, reverted MonetaryValue → Value)
reviewer-context: Corne (PM + lead) inline comment
created: 2026-05-15
purpose: Mechanical eval input for marketer-review skill / agent panel. Skill should mechanically identify the same failure pattern when given the original (pre-fix) code.
---

# Eval case: IsInvalid missing on Goal result

## What was the original code

Story 03 (`engage_get_goals` tool) originally produced a `GoalResult` record WITHOUT `IsInvalid`:

```csharp
// PRE-FIX (the failure state)
public record GoalResult(
    Guid Key,
    string Name,
    string GoalType,
    decimal MonetaryValue,    // separate failure — see pr-005-value-monetaryvalue.md
    bool IsMain,
    bool IsActive);
    // NO IsInvalid, NO IsInverted
```

Mapping in the tool's `ExecuteAsync` likewise omitted `IsInvalid`:

```csharp
var results = goals.Select(goal => new GoalResult(
    Key: goal.Key,
    Name: goal.Name,
    GoalType: _goalTypeFactory.GetGoalType(goal.GoalTypeId)?.Name ?? string.Empty,
    MonetaryValue: goal.Value,
    IsMain: goal.IsMain,
    IsActive: goal.IsActive
)).ToList();
```

The `IGoal` interface DOES expose `IsInvalid` (Engage Core `IGoal.cs`). The omission was deliberate — copy-pasted from Story 01 (`CampaignGroupResult`), which had dropped `IsInvalid` per its own Codex review.

## Why it failed

Pattern copy-paste without verifying the upstream filter behavior:

- **Story 01 (Campaigns):** the `CampaignGroup` repository auto-filters invalid groups at query level. So when the AI tool runs `ICampaignGroupService.GetAll()`, invalid groups never reach it. `IsInvalid` field on the result record would always be `false` — redundant. Codex review of Story 01 correctly flagged it as redundant; Story 01 dropped it.

- **Story 03 (Goals):** the `Goal` repository does NOT auto-filter. Invalid goals (e.g., goals whose tracked node was deleted) flow through `IGoalService.GetAll()` and reach the AI tool intact. WITHOUT `IsInvalid` on the result record, an LLM cannot tell the marketer the goal is broken.

The Story 03 implementer assumed pattern reuse without verifying the assumption was valid for Goals. Corne flagged this in PR #5 inline review.

## What a marketer should ask (failure scenario without fix)

> Marketer: "How is the Newsletter Signup goal performing?"
>
> Tool (without IsInvalid): returns goal with `IsActive=true, CompletionCount=0`.
>
> LLM interprets: "Newsletter Signup is active but has 0 completions this month — try improving the form CTA."
>
> Reality: the Form node that triggers the Newsletter Signup goal was deleted last month. The goal is technically active but cannot fire. Marketer wastes time tweaking CTA when the actual fix is to recreate the form binding.

The wasted action is the cost of omitting `IsInvalid`.

## Audit question (v1.1)

**Q1 — Repository filter policy.** Does the repository for this entity auto-filter out invalid or inactive instances at query level?

- For Goals: **NO.** Therefore `IsInvalid` MUST be surfaced.
- Pattern copy-paste from Story 01 is invalid because Campaigns repo behaves differently.

## Expected reviewer output

A marketer-perspective reviewer (human or AI) examining this code should produce:

```
FINDING — Q1 violation (missing IsInvalid on Goal result)

Severity: HIGH — blocks marketer from diagnosing broken goal configurations.

Evidence:
- IGoal.IsInvalid exists in Engage Core (Goal.cs, line 35-44 in original)
- IGoalService.GetAll() does NOT pre-filter by IsInvalid
- GoalResult record omits IsInvalid field
- Tool description does not mention validity

Comparison anti-pattern: Story 01 (CampaignGroupResult) correctly dropped IsInvalid because Campaigns repo auto-filters. Story 03 (GoalResult) copy-pasted the pattern without verifying — Goals repo does NOT auto-filter, so the pattern does not transfer.

Required fix:
1. Add IsInvalid field to GoalResult record (bool)
2. Wire goal.IsInvalid in the mapping
3. Tool description should explain: "IsInvalid=true means the goal's tracked object was deleted/broken; goal cannot fire even if IsActive=true."

Cross-reference: engage-workspace/notes/domain-rules.md — per-entity Q1 application matrix confirms Goal needs IsInvalid surfaced.
```

## How the fix landed (commit d483e05)

```csharp
// POST-FIX
public record GoalResult(
    Guid Key,
    string Name,
    string GoalType,
    decimal Value,            // renamed back from MonetaryValue (separate fix)
    bool IsMain,
    bool IsActive,
    bool IsInverted,          // added (separate fix — see pr-005-is-inverted.md)
    bool IsInvalid);          // added

// Tool description excerpt:
//   "...and IsInvalid (the goal's configuration is broken, e.g. it references a deleted node, so it cannot fire)."
```

## Mechanical eval criterion

When this eval case is fed to a marketer-review skill/agent:

**PASS:** the skill flags `IsInvalid` as missing AND cites Q1 (repository filter policy) AND notes the copy-paste anti-pattern from Story 01 (Campaigns) AND recommends the correct fix.

**FAIL:** the skill misses any of the above, OR incorrectly suggests the pattern is fine (echoing the original implementer's assumption).

## Related

- `feedback_domain_field_audit.md` v1.1 Q1
- `engage-workspace/notes/domain-rules.md` — Goal entity row in per-entity Q1 matrix
- `pr-005-is-inverted.md` (separate Q2 failure in same PR)
- `pr-005-value-monetaryvalue.md` (separate Q3 failure in same PR)
