---
title: Orchestrator design notes (deferred)
status: capture-only — these are decisions for the orchestrator phase, not personas
created: 2026-05-16
relates-to: personas/*.md (panel personas), this workspace's README.md
---

# Orchestrator design notes

This file captures architectural decisions that surface during persona drafting but belong to the **orchestrator layer** — not any individual persona. The orchestrator is the synthesis component that dispatches the panel, collects findings, merges, and produces a panel review. It does not exist yet; these notes accumulate as decisions emerge so the orchestrator design phase can refer back to them.

---

## 1. Compound severity rule

**Source:** Andy decision 2026-05-16 (persona #1 v0.2 ship verdict, severity reading choice).

**Decision:** Severity is per-finding standalone at the persona layer. Each persona scores Severity assuming the finding is the only issue in the diff. Cross-finding compounding is the **orchestrator's** job, not the persona's.

**What the orchestrator must implement:**

When two or more findings reference related Evidence in the same diff (same file, same record, semantically reinforcing wire-format claim), the orchestrator may elevate the panel-level severity above any individual finding's severity. Concrete cases:

- Two Concerns on related Evidence that compound a single fabricated-claim risk → panel-level **Block** flag, even though neither finding alone justified Block.
- A Block finding plus a Concern on the same Evidence → panel-level Block (no change), but the Concern's Recommendation should be folded into the Block's remediation plan.
- Multi-persona agreement (2+ personas flag overlapping Evidence with different framework citations) → orchestrator marks "high-confidence finding" and may elevate severity if individual calls were borderline.

**What this avoids:** personas reasoning about compound effects within their own output, which is the same drift as cross-persona reasoning — both contaminate the per-finding signal.

**Open design questions for the orchestrator phase:**

- Define "related Evidence" precisely (same file + same record name? token overlap threshold? framework-citation pair?).
- Threshold for Concern → Block elevation (any two Concerns on same Evidence, or requires specific pattern match?).
- Output shape — does the orchestrator emit a separate "panel-level severity flag" alongside the original per-finding severity, or rewrite the finding's severity? Recommend the former (preserves persona output integrity).

---

## 2. AI-reviewer-introduced origin stamping

**Source:** verdict on eval criterion #4 split (`eval-inputs/pr-005-value-monetaryvalue.md` addendum).

**Decision:** Origin attribution ("this failure was introduced by an AI reviewer") lives at the orchestrator layer, not the persona layer. Personas review the diff alone; the orchestrator stamps origin metadata by matching findings' Evidence strings against PR review history.

**What the orchestrator must implement:**

- Fetch PR review history via `gh CLI` (or equivalent) for the PR under review.
- For each finding produced by the panel, attempt to match its Evidence string against the change's origin in the PR comment thread / review suggestion log.
- Tag finding with `origin: implementer-authored | ai-reviewer-suggested | human-reviewer-suggested | unknown`.
- Surface AI-reviewer-suggested findings prominently in the panel summary (these are the highest-leverage catches because they prevent AI-reviewer-introduced bugs from shipping).

**Why this is deterministic, not reasoning:** the orchestrator does not infer origin from stylistic heuristics. It matches Evidence strings against actual PR data. If no match, tag is `unknown` — never speculation.

---

## 3. Persona Context-gap escalation routing

**Source:** persona #1 v0.2 "Overlap on a single Evidence" clarifier (audit-dimensions section).

**Decision:** When a persona finds Evidence that carries both its own dimension AND another persona's dimension, it flags only its own and emits a one-line `Context gap: <other-dimension> also applies on <Evidence>` at the end of its response.

**What the orchestrator must implement:**

- Parse `Context gap:` lines from each persona's response.
- Route the gap to the owning persona — either as a second-pass dispatch (re-invoke the owning persona with the Evidence highlighted) or by merging the gap note into the owning persona's existing finding on the same Evidence if one exists.
- Surface unresolved gaps (no owning persona produced a finding for the routed Evidence) in the panel summary as "potential coverage hole."

---

## 4. Dispatch parallelism + persona isolation

**Source:** session decisions 2026-05-16 (subagent dispatch via Agent tool, no shared memory between personas).

**Decision:** All four personas dispatch in parallel from the orchestrator session. Each subagent receives only its persona file + the diff + any orchestrator-injected metadata (e.g., PR title, review history excerpts if the orchestrator pre-fetches). No subagent sees another's output.

**What the orchestrator must implement:**

- Single message to the harness with 4 parallel Agent tool calls (per the Agent tool's parallel-execution guidance).
- After all 4 return, synthesis runs in the orchestrator session — not delegated to a 5th subagent (synthesis benefits from the orchestrator's full context including PR history + per-persona output).

---

## 5. Q5 forward-looking trigger criterion (deferred eval)

**Source:** Andy decision 2026-05-16 (persona #4 scope choice — option Y with explicit-acknowledgment enhancement).

**Decision:** Q5 evidence tier ships in persona #4's lane but without a synthetic positive eval. Engage currently has no `AttributionType` field surfaced through tools, so Q5 violations cannot be tested against real production code. Building a synthetic Q5 eval would compound the anchor-leak risk already flagged on persona #3's synthetic Q7 eval.

**Trigger criterion for creating a Q5 eval case in the future:** "Create Q5 eval case if Q5 catches diverge or appear inconsistent across 2+ orchestrator runs."

Concrete triggers:
- Persona #4 flags a Q5 issue and another persona (e.g., Brand-Voice on Q1(c) Stone-vs-Opinion) reads the same Evidence differently.
- Persona #4 emits `Context gap (need-context):` on Q5 classification across 2+ orchestrator runs (cannot decide if a count field is Observed / Modeled / Attributed without context).
- A real PR introduces an `AttributionType` field or evidence-tier metadata on a tool result record and Q5 classification becomes load-bearing for the panel review.

**What the orchestrator must implement when the trigger fires:**

- Surface the divergence or context-gap pattern in the panel review summary.
- Route to the panel-design layer (this workspace) for Q5 eval case creation.
- Until then: Q5 stays in persona #4's lane as a vigilance dimension; no orchestrator-side validation gate on Q5 catches.

---

## Maintenance

Add new entries here as decisions emerge during persona drafting. When the orchestrator design phase begins (after all 4 personas are drafted + validated), this file becomes input to the orchestrator spec.
