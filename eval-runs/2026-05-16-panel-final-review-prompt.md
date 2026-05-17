# Panel final review prompt — cross-cutting integration audit

Copy everything below the `---` line into a fresh Claude Code session at `D:/source/work/marketer-agents`. The session has read access to all referenced paths.

---

I've completed a 4-persona marketer-review agent panel for Engage.AI PRs. Each persona was validated individually in scope (catches its target eval case from PR #5, or 2-run negative + synthetic positive when no PR #5 eval covers its dimension). Before the orchestrator phase — which dispatches all 4 personas in parallel, synthesizes findings, and stamps origin metadata — I want a cross-cutting integration audit of the panel as a whole.

**Don't re-validate individual personas. Don't draft an orchestrator.** This is panel-level review only. Verdict + concrete actions, structured per question below.

## Files to read

In order:

1. `D:/source/work/marketer-agents/README.md` — panel overview, current status, validation table.
2. The 4 persona files (final versions):
   - `personas/01-brand-voice-marketer.md` (v0.3 — Q3 + Q1(c))
   - `personas/02-funnel-stage-marketer.md` (v0.2 — Q2 + Q6)
   - `personas/03-hypothesis-driven-marketer.md` (v0.1 — Q7)
   - `personas/04-data-trust-marketer.md` (v0.1 — Q1 + Q5 forward-looking)
3. The dispatch run outputs (for voice differentiation + anchor-leak evaluation):
   - `eval-runs/2026-05-16-persona-01-v0.2-pr005-monetaryvalue.md` (Brand-Voice's final dispatch — v0.3 added one rubric line but content is v0.2's)
   - `eval-runs/2026-05-16-persona-02-v0.1-pr005-isinverted.md` (Funnel-Stage — v0.2 changelog reflects Context gap unification only, dispatch text unchanged)
   - `eval-runs/2026-05-16-persona-03-v0.1-validation.md` (Hypothesis-Driven, both negative + synthetic positive runs)
   - `eval-runs/2026-05-16-persona-04-v0.1-pr005-isinvalid.md` (Data-Trust)
4. `notes/orchestrator-design-notes.md` — 5 deferred decisions accumulated during persona drafting.

Optional context if needed:
- `eval-inputs/` (3 PR #5 eval cases — what the panel must catch)
- `eval-runs/2026-05-16-review-prompt.md` (the persona #1 review prompt from earlier — gives you the precedent for structured review output)

## Context you should know that the files don't state

- The panel uses **subagent dispatch via the `Agent` tool**. Each persona file is a complete system prompt; subagents have no shared memory; orchestrator runs in a Claude Code session.
- **Dedupe + compound severity = orchestrator's job, not the persona's.** Personas review independently. Multi-persona agreement = confidence signal at orchestrator layer.
- **Context gap mechanism** was unified mid-build (cases A overlap / B need-context / C out-of-lane). Persona #1 (v0.3) and #2 (v0.2) got the unified mechanism via patch; personas #3 and #4 had it from day one.
- **Calibration-anchor leakage** was caught and fixed on persona #1 (v0.1 → v0.2 removed the anchor). Persona #3 has a Worked Example A whose shape is close to its synthetic positive validation diff — anchor-leak watch was flagged but not resolved. I want this audited.
- **Q5 eval was explicitly deferred** per a decision documented in `orchestrator-design-notes.md` §5. Persona #4 ships with Q1 validated and Q5 forward-looking. Trigger criterion for future Q5 eval is documented.
- **Original validation gate ("catch ALL 3 PR #5 eval cases")** is closed. Personas #1, #2, #4 each catch their respective eval; persona #3 has no PR #5 eval and was validated via 2-run pattern.

## Questions I want you to answer

### Q1 — Cross-persona consistency

Read all 4 persona files. Audit consistency across:
- **Severity rubric language.** Each persona inlines the Block / Concern / Nit definitions verbatim per the reviewer's earlier recommendation (no shared file). Are the definitions actually identical in structure, or did inlining drift? Specifically check: the "Severity is per-finding standalone" clarifier, the Block-trigger phrasing, the "if you find yourself wanting to mark Block, check…" tail sentence.
- **Output format template.** Compare the finding template across all 4 personas. Are the field labels (`Severity`, `Evidence`, `Framework citation`, `Issue`, `Recommendation`) and the structural metadata expectations identical? Are the Issue-leads instructions phrased consistently ("Sentence 1 MUST lead with…")?
- **Context gap mechanism.** All 4 personas have the unified A/B/C section. Read them side by side. Are the prefixes and default-behavior instructions consistent? Specifically: does persona #4's Q5-specific Case B note (about divergence trigger) introduce inconsistency, or is it a clean specialization?

Verdict format per sub-area: "Consistent" / "Minor drift in: <specifics>" / "Material drift requiring re-alignment: <specifics>". Then a one-line overall: "Consistency OK to proceed" or "Re-align before orchestrator."

### Q2 — Voice differentiation (blind test)

The four personas should sound distinct enough that a reader can identify which persona wrote a finding from voice alone. Test this:

1. From the dispatch run files, pick one finding from each persona (Brand-Voice Finding 1, Funnel-Stage Finding 1, Hypothesis-Driven positive-validation Finding 1, Data-Trust Finding 1).
2. For each, isolate the Issue body (the 2-4 sentence narrative paragraph — strip the structured metadata).
3. Without naming the personas, evaluate whether the four Issue bodies sound clearly different in vocabulary, decision-frame, and consequence-type. Specifically:
   - Brand-Voice should foreground language / what the LLM repeats / fabricated semantic.
   - Funnel-Stage should foreground flow / direction / scope / handoff.
   - Hypothesis-Driven should foreground experimentation / falsifiability / predicted lift / baseline.
   - Data-Trust should foreground broken-but-hidden / diagnostic failure / data-source / fidelity.

Could you assign each Issue body to its persona blind, based on voice and decision-frame alone? Specifically call out: any persona whose voice is bleeding into another's vocabulary, any persona whose voice reads as the same flat LLM-review tone as the others. If voice differentiation fails, name the offending persona and propose what tightening is needed.

### Q3 — Anchor-leak status (specifically persona #3)

Persona #3's Worked Example A inside its prompt (`personas/03-hypothesis-driven-marketer.md`) is *"If completion rates are below 5%, suggest testing alternative landing pages or revising the call-to-action copy."* The synthetic positive validation diff used different wording (a segments tool description about "test alternative messaging or pause spend"), but the structural pattern (description authorizes vague suggestion without hypothesis-shape) is very similar.

Read the persona #3 positive validation output (`eval-runs/2026-05-16-persona-03-v0.1-validation.md`). Evaluate:
- Did the dispatch findings echo verbatim phrases or structural cadence from Worked Example A?
- Or did they produce genuinely novel framing rooted in the framework alone?

The persona #1 v0.1 dispatch echoed calibration-anchor wording almost verbatim — that was the smoking gun that drove the v0.2 anchor removal. Apply the same test here: is persona #3's Worked Example A acting as a covert calibration anchor, or is the wording difference between example and synthetic diff enough to confirm framework-driven catches?

Verdict: "No leak — ship as-is" / "Suspect leak — reduce Worked Example A specificity before orchestrator" / "Confirmed leak — Worked Example A must be replaced or removed."

### Q4 — Orchestrator-readiness check

Read `notes/orchestrator-design-notes.md`. Five entries accumulated:
1. Compound severity rule (Concern × N → panel-level Block).
2. AI-reviewer-introduced origin stamping (orchestrator matches Evidence to PR review history).
3. Persona Context-gap escalation routing (orchestrator parses `Context gap:` lines and routes).
4. Dispatch parallelism + persona isolation (4 parallel Agent calls, synthesis in orchestrator).
5. Q5 forward-looking trigger criterion (when to create a Q5 eval).

Then read the panel design (README + persona files) and ask: **does the orchestrator have everything it needs to dispatch the panel, synthesize findings, and produce a panel-level review?** Identify gaps. Specifically check:

- Is there a documented expected input shape for the orchestrator (PR diff + PR review history + ...)?
- Is the synthesis output shape defined (or at least sketched — what does the orchestrator produce after merging 4 personas' findings)?
- Multi-persona agreement handling — orchestrator-design-notes §1 mentions "multi-persona agreement → orchestrator marks high-confidence finding" but the merge mechanics (how to compare Evidence strings, how strict matching is) aren't spec'd.
- Failure modes — what does the orchestrator do if a persona's subagent times out, returns malformed output, or hits the no-findings sentinel for all four personas?
- Error in Context gap routing — what if a Case C out-of-lane catch points to a dimension no persona owns (e.g., Q4 decision-class, currently out of panel scope)?

Output: bulleted list of "Documented" / "Partially documented — needs detail" / "Gap — must spec before orchestrator phase". Triage which gaps are blocking the orchestrator phase vs which can be filled during the orchestrator design itself.

### Q5 — Severity rubric application across personas

Each persona has its own Block / Concern / Nit examples specialized to its dimensions. Looking at the actual dispatched outputs:

- Brand-Voice: Finding 1 Block (MonetaryValue rename), Finding 2 Block (description prose leak) — note that v0.1 reviewer argued Finding 2 should be Concern under standalone reading; Andy decided standalone (option A); v0.2 dispatch produced Block. Outcome with v0.2 rubric: still Block. Was that the right outcome, or did the v0.2 rubric edit fail to enforce the standalone reading?
- Funnel-Stage: Finding 1 Block (IsInverted missing), Finding 2 Concern (handoff missing).
- Hypothesis-Driven synthetic: Findings 1+2+3 all Block.
- Data-Trust: Finding 1 Block (IsInvalid missing), Finding 2 Concern (data-source fidelity).

Are these severity calls consistent across personas, given each persona's rubric uses different example shapes? Specifically: are personas calling Block on substantively comparable failure modes, or is one persona over-claiming Block while another under-claims? The persona #1 reviewer warned about Block-drift — has it crept back in any persona, especially after the v0.2 standalone-reading edit?

Verdict per persona: "Severity calls correctly calibrated" / "Suspect over-claim" / "Suspect under-claim" — with one-sentence specific example each.

### Q6 — Final ship verdict

After Q1-Q5: is the panel ready to proceed to orchestrator phase?

- **Ship panel as-is.** Proceed to orchestrator design with no persona changes. All gaps captured in notes are acceptable for the orchestrator design phase to spec.
- **Iterate to v0.X.** Specific changes per persona needed before orchestrator phase. Enumerate them concretely (which file, which section, what to change).
- **Significant rework.** A structural issue across the panel that requires re-validating one or more personas. Name the issue and the minimal recovery path.

If "Ship": briefly call out the top 2-3 architectural risks the orchestrator phase will need to mitigate, so I can prioritize them when I scope orchestrator work.

## Output format I want

Structured markdown response, section per question above. Lead each section with a one-line verdict, then 4-8 sentences of justification. End with an `## Overall verdict` section that picks one of the three options in Q6 and lists the top 3 actions (if any) before orchestrator phase.

Be direct. Hedging slows me down. I'd rather hear "voice on persona #3 is bleeding into persona #1's territory because…" than "voices are mostly distinct but…"

Don't summarize the panel design. Don't praise. Critique.
