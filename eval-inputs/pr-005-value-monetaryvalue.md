---
title: PR #5 eval — MonetaryValue rename treated Opinion-class field as Stone-class
id: pr-005-value-monetaryvalue
case-type: failure-prevention
audit-question: Q3 — Generic vs imposed semantic (Stone vs Opinion)
source-pr: umbraco/umbraco.engage.ai PR #5 (Story 03 — engage_get_goals)
source-fix: commit d483e05 (reverted MonetaryValue → Value)
reviewer-context: Corne (PM + lead) inline comment correcting Codex round 1 suggestion
created: 2026-05-15
purpose: Mechanical eval input for marketer-review skill / agent panel. Tests the AI-suggested-semantic-imposition failure mode — where an AI reviewer (Codex in this case) renames a generic field to a specific semantic that the data does not guarantee.
notable: This is the failure mode where AI reviewers (Codex / pr-review-toolkit) actively introduced the bug. Stone-vs-Opinion is the precise OSS discipline that prevents it.
---

# Eval case: MonetaryValue rename — Opinion-class field promoted to Stone-class

## What was the original code

Story 03 (`engage_get_goals` tool) initially had `GoalResult.Value` (matching `Goal.Value` from Engage Core):

```csharp
// EARLIEST DRAFT (before Codex round 1)
public record GoalResult(
    Guid Key,
    string Name,
    string GoalType,
    decimal Value,            // matches Goal.Value naming
    bool IsMain,
    bool IsActive);
```

After Codex round 1 review (during Story 03 implementation), Codex suggested renaming `Value` → `MonetaryValue` "for LLM clarity". The implementer accepted the suggestion:

```csharp
// AFTER CODEX ROUND 1 — INTRODUCED THE FAILURE
public record GoalResult(
    Guid Key,
    string Name,
    string GoalType,
    decimal MonetaryValue,    // renamed from Value — Codex suggestion accepted
    bool IsMain,
    bool IsActive);
```

The PR #5 review surfaced this rename as a domain error. Corne reverted it back to `Value` in commit d483e05.

## Why it failed

`Goal.Value` is a **generic decimal** that the marketer configures with their own meaning. Engage does not enforce a semantic:

| Marketer configures `Goal.Value` as... | Meaning intended |
|---|---|
| `10.00` | $10 average pipeline value per signup (monetary) |
| `1` | Count this as 1 conversion (count) |
| `250` | $250 customer LTV (monetary) |
| `5` | 5 priority points in our scoring system (score) |
| `0.5` | Half-weight goal in scoring (weight) |
| `0` | Tracking only, no business value (sentinel) |

The marketer's intent depends on configuration. Engage's Q3 design choice: keep the field generic, let marketer impose meaning per goal.

Codex's rename `MonetaryValue` violated this:

- Forced an interpretation (money) that the data does not guarantee
- LLM seeing `MonetaryValue=5.50` would say "Newsletter Signup is worth $5.50" — fabricated currency claim
- Aggregation (sum `MonetaryValue` across goals) becomes meaningless if Values aren't money
- Marketer reading the LLM output gets misled about their own configuration

This is the **Opinion-treated-as-Stone** failure mode (precise OSS name from `digital-marketing-pro/stone-vs-opinion.md`):

- **Stone-class field:** system-validated truth (e.g., `Goal.IsActive` from repository — always reflects DB state)
- **Opinion-class field:** marketer-asserted intent (e.g., `Goal.Value` configured semantic — marketer decides meaning)

Codex renamed an Opinion-class field as if it were Stone-class. The data layer did not change; only the wire-format name changed. But the rename signals to the LLM "this is money" — a meaning the data does not guarantee.

## What a marketer should ask (failure scenario without fix)

> Marketer: "What's the total value of all our active goals this quarter?"
>
> Tool (with MonetaryValue): returns goals with `MonetaryValue: 10.00, 5.00, 250.00, 0.50, 0.00`.
>
> LLM interprets: "Total monetary value of your active goals = $265.50."
>
> Reality: marketer configured the Values as a MIX of currency, scores, and weights. The "$265.50" total is meaningless arithmetic — adding apples (dollars) + oranges (priority points) + grapes (weights) and calling the sum a dollar amount.

The wasted action: marketer makes a business decision based on a fabricated dollar figure.

## Audit question (v1.1)

**Q3 — Generic vs imposed semantic.** Is the field a generic decimal/string/int that the marketer configures with their own meaning?

- For `Goal.Value`: **YES — generic decimal, marketer-configurable.**
- DO NOT rename to a specific semantic.
- Use a generic name and let the tool description note configurability.

**Q1 extension — Stone vs Opinion semantic class per field.**

- `Goal.Value` is **Opinion-class** (marketer-asserted intent).
- AI reviewers (Codex, pr-review-toolkit) must NOT silently promote it to Stone-class semantic via rename.
- Field-level enum `SemanticClass: Stone | Opinion` on the record is the structural fix.

## Expected reviewer output

A marketer-perspective reviewer examining the renamed code (with `MonetaryValue`) should produce:

```
FINDING — Q3 violation + Q1(c) violation (Opinion-treated-as-Stone)

Severity: HIGH — wire-format name imposes semantic on data that doesn't guarantee it.
Failure class: AI-reviewer-introduced. The bug was created by Codex's suggestion, not the implementer's choice.

Evidence:
- Goal.Value is decimal with no schema-enforced semantic (Engage Core IGoal.Value)
- Marketers in production configure Value as a mix of money / count / score / weight / zero
- Rename to MonetaryValue forces money interpretation
- LLM consuming "MonetaryValue" field will report findings as currency, fabricating semantic the data does not back

Stone-vs-Opinion classification:
- Goal.Value = Opinion-class (marketer asserts meaning per goal)
- Rename to MonetaryValue treats it as Stone-class (system-validated money) — wrong

Required fix:
1. Revert MonetaryValue → Value
2. Tool description explains: "Value is a marketer-configured decimal whose meaning depends on goal setup (monetary, count, score, weight, or zero for tracking-only). Do not assume currency."
3. Add SemanticClass field-level enum to record (future architectural improvement): SemanticClass.Opinion on Value, SemanticClass.Stone on system-tracked fields.

Cross-reference:
- feedback_domain_field_audit.md v1.1 Q3 + Q1(c)
- engage-design-patterns.md §2 (Stone vs Opinion shape variants)
- This case is the canonical PR-#5-class failure that motivated the v1.1 audit rule expansion.
```

## How the fix landed (commit d483e05)

```csharp
// POST-FIX
public record GoalResult(
    Guid Key,
    string Name,
    string GoalType,
    decimal Value,            // reverted from MonetaryValue
    bool IsMain,
    bool IsActive,
    bool IsInverted,
    bool IsInvalid);

// Tool description excerpt:
//   "Value is a marketer-configured decimal whose meaning depends on the goal's setup —
//    it might represent monetary value, a count, a score, a weight, or be left at zero for tracking-only goals.
//    Do not assume currency."
```

## Mechanical eval criterion

When this eval case is fed to a marketer-review skill/agent:

**PASS:** the skill flags the `MonetaryValue` rename as a Q3 / Stone-vs-Opinion violation AND cites the OSS pattern name (Opinion-treated-as-Stone) AND recommends revert to `Value` AND surfaces that the failure was introduced by an AI reviewer (not the implementer).

**FAIL:** the skill agrees with the rename (echoes Codex's original reasoning), OR misses that Goal.Value is configurable, OR recommends a different specific semantic (`ScoreValue`, `PointValue`) instead of generic `Value`.

## Eval criteria — split per panel layer (marketer-agents workspace v0.2)

This addendum (added 2026-05-16 by Andy + reviewer verdict at `eval-runs/2026-05-16-review-prompt.md`) splits the PASS criteria above into the layer that should produce each signal. The criteria do not change; only the responsibility assignment does.

**Persona-level PASS** (a single persona reviewing the code diff alone — what `personas/01-brand-voice-marketer.md` is responsible for):

1. Flag the `MonetaryValue` rename as a Q3 / Stone-vs-Opinion violation.
2. Cite the OSS pattern name (Opinion-treated-as-Stone).
3. Recommend revert to `Value` AND propose tool-description language that surfaces configurability without imposing semantic.

If the persona produces all three, persona-level PASS is achieved. The persona is *not* expected to produce signal #4 — origin attribution lives in PR review history, not in the code diff.

**Panel-level PASS** (the full panel orchestrator — persona findings + PR review history metadata):

4. Persona-level PASS achieved, AND
5. Orchestrator stamps `origin: ai-reviewer-suggested` on the relevant finding by matching its Evidence string against PR review history (Codex round-1 suggestion in this case). The origin tag is a deterministic match against PR data, not a reasoning step.

**Why the split:** A persona reviewing only a code diff has no reliable signal for "this was AI-reviewer-introduced." That signal lives in commit messages, PR comments, and Codex/pr-review-toolkit suggestion records — outside the diff. Asking the persona to infer origin from stylistic heuristics ("overly verbose naming," "for LLM clarity" rationale shapes) contaminates framework-grounded findings with speculation. The orchestrator owns the deterministic match.

**FAIL conditions unchanged:** as listed above. They apply at persona-level (the persona's three responsibilities).

## Notable meta-finding

This eval case is special because the failure was **introduced by an AI code reviewer**, not by the implementer. It is the primary motivation for the v1.1 audit rule:

- v1 audit (3 questions) would have caught this via Q3 — but only if applied at planning phase
- The failure happened DURING Codex review, when Q3 was not in Codex's context
- v1.1 + the Codex prompt template (`engage-workspace/notes/domain-rules.md`) now require Stone-vs-Opinion declarations to be included in Codex prompts — preventing Codex from making semantic-imposing suggestions blind to domain context

Marketer-review skill / agent panel should be calibrated to catch this failure class specifically because AI reviewers are the most likely source.

## Related

- `feedback_domain_field_audit.md` v1.1 Q3 + Q1(c) Stone-vs-Opinion
- `engage-workspace/notes/engage-design-patterns.md` §2 (Stone vs Opinion shape variants)
- `pr-005-is-invalid.md` (separate Q1 failure in same PR)
- `pr-005-is-inverted.md` (separate Q2 failure in same PR)
- D:/source/work/marketer-agent-research/notes/patterns/domain-audit-equivalents.md (cross-source verification of Stone-vs-Opinion as recurring OSS pattern)
