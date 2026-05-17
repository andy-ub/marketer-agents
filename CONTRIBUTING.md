# Contributing

Short version: this is an experimental repo. PRs welcome for new personas, new eval cases, orchestrator fixes, and clearer docs. **New personas must ship with a passing eval case.**

---

## Issues

Useful issue types:

- **Bug:** the orchestrator misbehaves (parse failure, wrong merge, wrong severity elevation, malformed output not retried, etc.).
- **Persona suggestion:** a new audit dimension that catches a class of bug the current 4 personas miss. Open an issue first with a real or synthetic example — adding a persona without a real failure pattern in hand tends to produce over-generic prompts.
- **Eval case:** a failure pattern (with pre-fix code and PASS criteria) you'd like added to the calibration set.
- **Use-case question:** how to apply the panel to a non-Engage codebase, or whether a specific failure pattern fits an existing persona's lane.

---

## PRs

### New persona

Required artifacts:

1. **Persona file** `personas/0N-<role>-marketer.md` — follow the structural template established by [`personas/01-brand-voice-marketer.md`](personas/01-brand-voice-marketer.md). Sections in order: Identity → Framework → Worked examples → Audit dimensions you own → Severity rubric → Output format (with Voice — what to do / what NOT to do) → Constraints → Context gap mechanism → Reference.
2. **Eval case** `eval-inputs/<id>.md` — a code snippet (real PR or synthetic) where this persona's audit dimension catches a specific failure. Frontmatter + mechanical PASS/FAIL criteria.
3. **Live-dispatch validation run** in `eval-runs/` — output of the persona against its eval, with verdict.
4. **README update** — add the persona to the persona table.

Validation gate: the persona must catch its target eval case via live subagent dispatch (or pass a 2-run negative + synthetic positive validation if no real eval covers the dimension). Don't merge before the gate is closed.

### New eval case

Frontmatter must include: `title`, `id`, `case-type`, `audit-question` (which Qn), `source-pr` or `source: synthetic`, `created`, `purpose`. Body must include: pre-fix code, why it failed, expected reviewer output, mechanical PASS/FAIL criterion. Mirror an existing file in `eval-inputs/` for shape.

### Orchestrator fix

Document the bug in an `eval-runs/<date>-<bug-description>.md` analysis with: input that triggered the bug, observed output, expected output, root cause, proposed fix, before/after measurement (precision/recall for merge logic; severity-call delta for rubric; etc.). Reference the analysis from the commit message.

### Docs / clarity

Open PRs freely. Keep README under 250 lines; deep details belong in `HANDOFF.md` or `notes/`.

---

## Style

- Voice in persona Issue/Recommendation bodies must be consequence-first (what the LLM will say to the marketer / what resource the marketer will commit) — not schema-paraphrase. See the "Voice — what NOT to do" sections in any persona file for the anti-trope.
- Findings cite a framework dimension. Uncited findings get downgraded by the orchestrator.
- Severity is per-finding standalone — the orchestrator handles compound elevation. Don't reason about other findings when calling severity inside a persona.
- One persona owns each dimension. Cross-lane catches escalate via `Context gap (out-of-lane):` lines, not as findings.

---

## What this repo is NOT

- **Not a general-purpose code review tool.** It catches domain-semantic bugs in LLM-facing tool surfaces (records, descriptions, recommendation fields). For idiomatic correctness, performance, security, test coverage — use existing review tooling alongside it.
- **Not a replacement for human review.** The panel is a screen, not a verdict. Block findings deserve human attention before merge; the panel doesn't merge anything.
- **Not framework-specific.** The patterns generalize to any LLM-facing API where wire-format names/types/descriptions become claims the LLM narrates. Engage.AI is just where the calibration data lives.

---

## Author

Andy Vu Viet ([andy-ub](https://github.com/andy-ub)). PRs reviewed on weekends or when shipping Engage.AI work happens to surface related findings.
