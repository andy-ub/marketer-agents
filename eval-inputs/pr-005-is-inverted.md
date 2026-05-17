---
title: PR #5 eval — IsInverted missing on Goal result
id: pr-005-is-inverted
case-type: failure-prevention
audit-question: Q2 — Polarity semantic
source-pr: umbraco/umbraco.engage.ai PR #5 (Story 03 — engage_get_goals)
source-fix: commit d483e05
created: 2026-05-15
purpose: Mechanical eval input for marketer-review skill / agent panel. Tests the polarity-blindness failure mode — LLM interpreting metric direction without entity-level polarity context.
---

# Eval case: IsInverted missing on Goal result

## What was the original code

Story 03 (`engage_get_goals` tool) originally produced a `GoalResult` record WITHOUT `IsInverted`:

```csharp
// PRE-FIX
public record GoalResult(
    Guid Key,
    string Name,
    string GoalType,
    decimal MonetaryValue,
    bool IsMain,
    bool IsActive);
    // NO IsInverted, NO IsInvalid
```

The `IGoal` interface DOES expose `IsInverted` (Engage Core `IGoal.cs`). The omission was unconscious — the story spec only mentioned `IsMain` and `IsActive` as boolean fields to surface, and `IsInverted` was not flagged during planning.

## Why it failed

`IsInverted` is a polarity flag — it flips the interpretation of completion counts:

- **Normal goal:** more completions = better. Example: `Newsletter Signup` (IsInverted=false). High count = good marketer outcome.
- **Inverted goal:** more completions = worse. Example: `Cart Abandoned`, `Bounced`, `Form Started but Not Submitted` (IsInverted=true). High count = failure mode, not success.

Without surfacing `IsInverted`, an LLM seeing "Cart Abandoned: 487 completions this month" might celebrate it as top-performing instead of flagging it as a failure mode to investigate.

This is polarity-blindness: a metric reading is meaningless without direction context. Magnitude (487) alone tells you nothing about whether the marketer should be happy or worried.

## What a marketer should ask (failure scenario without fix)

> Marketer: "Which goals are performing best this month?"
>
> Tool (without IsInverted): returns all goals sorted by `CompletionCount` descending. Top result: `Cart Abandoned` with 487 completions.
>
> LLM interprets: "Cart Abandoned is your top-performing goal this month with 487 completions. Newsletter Signup is second with 23."
>
> Reality: `Cart Abandoned` is an inverted goal — 487 completions means 487 customers abandoned their cart. This is the marketer's worst-performing metric, not best. The LLM celebrated a failure mode as a success.

The wasted action: marketer either ignores the misleading celebration (wasting the tool call) OR worse, internalizes that high abandonment rates are good performance.

## Audit question (v1.1)

**Q2 — Polarity semantic.** Is there any boolean field on the entity that flips the interpretation of associated metrics?

- For Goals: **YES — `IsInverted`.** It flips metric direction.
- Surfacing it is REQUIRED. Tool description must explain the semantic.

Note from v1.1: polarity is novel Engage architecture — the marketer-agent-research survey found NO OSS source with entity-level polarity flags (7/9 absent). Engage's `IsInverted` framing is worth preserving.

## Expected reviewer output

A marketer-perspective reviewer (human or AI) examining this code should produce:

```
FINDING — Q2 violation (missing IsInverted on Goal result)

Severity: HIGH — LLM cannot interpret metric direction. Risk: celebrating failure modes as success.

Evidence:
- IGoal.IsInverted exists in Engage Core (Goal.cs)
- GoalResult record omits IsInverted field
- Tool description does not mention polarity / inverted goals
- A marketer asking "best-performing goal" would receive misleading interpretation if any inverted goal has high count

Polarity-blindness failure pattern: count alone is meaningless without direction. Magnitude × Polarity = interpretation. Without polarity, every count looks like success.

Required fix:
1. Add IsInverted field to GoalResult record (bool)
2. Wire goal.IsInverted in the mapping
3. Tool description must explain: "IsInverted=true means the goal counts a NEGATIVE outcome (e.g., bounce, cart abandonment). A high completion count is BAD for inverted goals, not good. LLM should interpret count direction accordingly."

Cross-reference: feedback_domain_field_audit.md v1.1 Q2; this is the textbook polarity-semantic case.
```

## How the fix landed (commit d483e05)

```csharp
// POST-FIX
public record GoalResult(
    Guid Key,
    string Name,
    string GoalType,
    decimal Value,
    bool IsMain,
    bool IsActive,
    bool IsInverted,          // added
    bool IsInvalid);          // added (separate fix)

// Tool description excerpt:
//   "...IsInverted (the goal counts a NEGATIVE outcome such as a bounce or abandoned cart —
//    a high completion count is bad, not good)..."
```

## Mechanical eval criterion

When this eval case is fed to a marketer-review skill/agent:

**PASS:** the skill flags `IsInverted` as missing AND cites Q2 (polarity semantic) AND explains the polarity-blindness failure mode (LLM celebrating failure as success) AND recommends adding the field with tool description explanation.

**FAIL:** the skill misses the polarity dimension entirely, OR only flags `IsInvalid` (different Q) without surfacing `IsInverted`, OR provides explanation but misses the LLM interpretation risk.

## Related

- `feedback_domain_field_audit.md` v1.1 Q2
- `engage-workspace/notes/domain-rules.md` — Goal entity is the canonical example of a polarity-bearing entity
- `pr-005-is-invalid.md` (separate Q1 failure in same PR)
- `pr-005-value-monetaryvalue.md` (separate Q3 failure in same PR)
