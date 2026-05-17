---
title: Persona-dispatch prompt template
status: v0.1
created: 2026-05-16
used-by: orchestrator/orchestrator-prompt.md Step 2
---

# Persona-dispatch prompt template

This is the boilerplate prompt the orchestrator sends to each subagent during `dispatch_personas_in_parallel` (Step 2). Substitute the curly-brace placeholders before sending. The template intentionally mirrors the pattern used during panel-design validation runs in `eval-runs/2026-05-16-*.md` — that pattern has been validated against all 4 personas.

---

## Template

```
You are taking on the role described in this persona file. Read it in full — it is your complete role brief, including framework, audit dimensions, severity rubric, output format, voice instructions, and constraints:

`{persona-file-absolute-path}`

After reading, review the following code diff from Engage.AI:

```{language-hint or omit}
{diff-text}
```

{optional-context-block}

Now produce your panel review of this diff. Apply ONLY your audit dimensions. Follow the output format in your persona file exactly — including the consequence-first Issue structure, the voice instructions, and the no-findings sentinel rule. Do not stretch on dimensions outside your lane.

Output only the findings (or the "No findings" sentinel). Nothing else.
```

---

## Placeholders

### `{persona-file-absolute-path}`

The full absolute path to the persona file being dispatched. Examples:
- `personas/01-brand-voice-marketer.md`
- `personas/04-data-trust-marketer.md`

### `{language-hint}`

Code-fence language hint based on the diff content. Default to `csharp` for Engage.AI tools; `diff` if a unified diff format with `+`/`-` markers; omit if mixed content.

### `{diff-text}`

The raw diff or pre-fix code snapshot. For:
- **Local form / pasted form:** the diff text the user provided or that `git show` produced.
- **PR-ref form:** the output of `gh pr diff` for the PR. If the diff is large (>200 lines), include only the relevant files — typically `**/Tools/*.cs` for Engage.AI tool changes — and note the truncation: `(diff truncated to Tools/ subtree; full diff omitted for length)`.

### `{optional-context-block}`

Used to inject orchestrator-known facts that personas can't derive from the diff alone. Common cases:

- **Source-of-truth references** for omission-class findings (e.g., interface declarations the diff doesn't show):
  ```
  Additional context about the underlying data layer:
  - `IGoal` in Engage Core (`Umbraco.Engage.Infrastructure.Analytics.Goals`) exposes `IsInverted` (bool) and `IsInvalid` (bool) on each goal instance.
  - `IGoalService.GetAll()` does NOT pre-filter by IsInvalid — invalid goals flow through.
  ```

- **Prior-story decision context** for Q1 copy-paste anti-pattern detection (Data-Trust persona specifically):
  ```
  Prior-story context for Q1 repository-filter-policy verification:
  - Story 01 (`engage_get_campaigns`) shipped a `CampaignGroupResult` record that intentionally OMITS `IsInvalid` — `ICampaignGroupService.GetAll()` auto-filters invalid groups. Surfacing was determined to be wire-format noise during Story 01's PR review.
  ```

- **PR title + body excerpt** for any persona, when PR metadata is available — helps personas understand the change intent. Limit to ~500 chars:
  ```
  PR title: <title>
  PR body excerpt:
  <first 500 chars of body>
  ```

If no useful context is available, omit this block entirely (do not emit empty `Additional context:` headers).

---

## Sub-prompt for failed-dispatch retry (Step 2 retry path)

When retrying a malformed-output persona once, use the same template verbatim. Do not add "the previous response was malformed, please try again" — that biases the subagent and may cause it to over-correct (e.g., re-emit the same content with cosmetic changes). One clean retry only.

---

## Concrete worked example (verifiable against the validation runs)

For the canonical PR #5 pre-fix dispatch on persona #1 (Brand-Voice Marketer), the substituted prompt was:

```
You are taking on the role described in this persona file. Read it in full — it is your complete role brief, including framework, audit dimensions, severity rubric, output format, voice instructions, and constraints:

`personas/01-brand-voice-marketer.md`

After reading, review the following Engage.AI code (the state of `src/Umbraco.Engage.AI/Tools/GetGoalsTool.cs` at commit `d483e05^`, i.e. *before* PR #5 review fixes were applied):

```csharp
[... pre-fix GetGoalsTool.cs source ...]
```

Additional context about the underlying data layer (you can verify this by reading the [Umbraco Engage Core source](https://github.com/umbraco/Umbraco.Engage) if needed): `Goal.Value` in Engage Core is a generic `decimal` property on `IGoal`. The Engage UI lets marketers configure it freely — it might represent a monetary amount, a count, a priority score, a weight, or be left at zero for tracking-only goals. The data layer does not enforce any specific semantic on this field.

Now produce your panel review of this diff. Apply ONLY your audit dimensions (Q3 generic-vs-imposed, Q1(c) Stone-vs-Opinion). Follow the output format in your persona file exactly — including the consequence-first Issue structure and the voice instructions. Do not summarize or editorialize outside the finding template. If you find no Q3 or Q1(c) issues, respond with the exact "No findings" sentinel from the persona file.

Output only the findings (or the "No findings" sentinel). Nothing else.
```

This dispatch produced the v0.2 validation output that closed PASS 3/3 against `pr-005-value-monetaryvalue.md`. The orchestrator-generated dispatch should match this structure precisely; deviations are bugs.
