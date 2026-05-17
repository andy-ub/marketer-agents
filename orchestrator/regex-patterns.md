---
title: Regex parsing cheatsheet for persona outputs
status: v0.1
created: 2026-05-16
used-by: orchestrator/orchestrator-prompt.md Step 3 (parse_findings + parse_context_gaps)
---

# Regex parsing cheatsheet

Patterns the orchestrator uses to parse persona subagent outputs. These are reference patterns — actual matching is done in-session by Claude reading the persona response and extracting structured data, but using these patterns as the authoritative shape definitions.

When matching, use multiline mode (PCRE `m` or Python `re.MULTILINE`) and DOTALL where indicated. All patterns are PCRE-compatible.

---

## §1 — Finding template

A persona finding consists of:

- A markdown H3 header starting with `### Finding ` followed by a number and a title.
- A bullet list with exactly 5 fields, in this order: Severity / Evidence / Framework citation / Issue / Recommendation.
- Each field's value may span multiple lines until the next bullet or the next finding header.

### Header pattern

```
^### Finding\s+(\d+):\s+(.+?)$
```

Captures: group 1 = finding number, group 2 = title.

### Per-field patterns (apply DOTALL within the finding's body span)

**Severity:**
```
^\s*[-*]\s+\*\*Severity:\*\*\s+(Block|Concern|Nit)\s*$
```

**Evidence:**
```
^\s*[-*]\s+\*\*Evidence:\*\*\s+(.+?)(?=^\s*[-*]\s+\*\*[A-Z]|^### Finding|^Context gap|\Z)
```

The lookahead terminates at the next bullet field, the next finding header, a Context gap line, or end of input.

**Framework citation:**
```
^\s*[-*]\s+\*\*Framework citation:\*\*\s+(.+?)(?=^\s*[-*]\s+\*\*[A-Z]|^### Finding|^Context gap|\Z)
```

**Issue:**
```
^\s*[-*]\s+\*\*Issue:\*\*\s+(.+?)(?=^\s*[-*]\s+\*\*[A-Z]|^### Finding|^Context gap|\Z)
```

**Recommendation:**
```
^\s*[-*]\s+\*\*Recommendation:\*\*\s+(.+?)(?=^### Finding|^Context gap|\Z)
```

### Validation rule

A finding is considered well-formed iff all 5 fields are present and the Severity value is exactly `Block`, `Concern`, or `Nit`. Missing any field → treat persona output as malformed (Step 2b retry path).

---

## §2 — No-findings sentinels (per persona)

Each persona has a specific sentinel string. The full response must equal the sentinel (allow surrounding whitespace and a trailing Context-gap block — the persona is permitted to emit a sentinel + a Context-gap escalation if it noticed something Case C-worthy).

Sentinel strings:

```
Brand-Voice (01):       No Q3 or Q1\(c\) findings\.
Funnel-Stage (02):      No Q2 or Q6 findings\.
Hypothesis-Driven (03): No Q7 findings\.
Data-Trust (04):        No Q1 or Q5 findings\.
```

### Sentinel + Context-gap pattern

Allow:

```
^\s*(No Q[0-9()c ]+ or Q[0-9()c ]+ findings\.|No Q7 findings\.)\s*$
(optionally followed by one or more Context gap lines — see §3)
```

### Detection logic

1. Check if the response (after trimming whitespace) starts with one of the sentinel strings and contains no `### Finding` headers.
2. If yes → record `persona_findings = []` and proceed to Context-gap extraction (§3).
3. If no → run §1 finding extraction.

---

## §3 — Context-gap lines

Context gap lines may appear:

- After the last finding (most common).
- After a no-findings sentinel.
- Mixed inside a sentinel-only response.

### Pattern

```
^Context gap(?:\s+\((overlap|need-context|out-of-lane)\))?:\s+(.+?)$
```

Captures:
- group 1 = case label (`overlap` | `need-context` | `out-of-lane`). Group is optional — older persona outputs may have emitted just `Context gap:` without the case prefix. Treat missing case as `case = "unspecified"` and surface in `routed_gaps` with resolution = `unresolved — case prefix missing, escalate to panel-design`.
- group 2 = the remainder of the line (the dimension reference and any commentary).

### Multi-line gap notes

If a gap spans multiple lines (rare but possible — e.g., persona elaborates the gap reason), terminate at the next `Context gap` line, the next `###` header, or EOF.

---

## §4 — Token extraction (anchor + context, post Bug §1 fix)

For each finding, extract **two** token sets — anchor tokens (primary signal for related-finding detection) and context tokens (fallback signal). These drive related-Evidence grouping in Step 4a.

**Why two sets:** the smoke-test on 2026-05-17 ([`eval-runs/2026-05-17-orchestrator-smoke-test-pr005.md`](../eval-runs/2026-05-17-orchestrator-smoke-test-pr005.md) Bug §1) showed that using Evidence-backtick tokens alone over-merges 18 of 21 pairs (86% false-merge rate). Root cause: omission-class findings cite the full record declaration as context, which contains *other fields' names* as token noise. Andy's initial stopword + ≥2-token proposal proved structurally insufficient — `MonetaryValue` token appears in both BV-F1's Evidence (rename finding) and FS-F1's Evidence (the record declaration with IsInverted missing), so they share 3+ specific tokens despite being independent findings. Title-anchor approach addresses the root cause: a finding's title names what the finding is *about*, which the Evidence backtick content often does not.

### Anchor tokens — extracted from finding title

The finding title is everything after `### Finding N:` up to the newline. Algorithm:

1. Strip the `### Finding N:` prefix and any em-dash-separated qualifier ("Polarity flag dropped on the wire — inverted goals will be celebrated as top performers" → both halves are anchor source).
2. Split on whitespace, then on camelCase / dots / hyphens / underscores (same boundaries as before).
3. Lowercase all tokens.
4. Drop tokens shorter than 3 characters.
5. Apply the **aggressive stopword list** (below). Result is the anchor token set.

### Context tokens — extracted from Evidence backticks (legacy behavior)

1. Find all substrings inside backticks (`` `...` ``) in the Evidence field.
2. Split + lowercase + length filter (same boundaries as anchor extraction).
3. Apply the **standard stopword list** (below).

Result is the context token set. Used only as a fallback when anchor tokens and framework citations are both inconclusive.

### Stopword lists

**Standard list** (applied to both anchor and context):

- English: `the`, `and`, `for`, `with`, `this`, `that`, `from`, `into`, `each`, `our`, `their`, `are`, `was`, `will`, `not`, `does`, `has`, `have`, `had`, `did`, `but`, `nor`, `you`, `out`, `who`, `whom`, `whose`, `which`, `what`, `when`, `where`, `why`, `how`, `any`, `all`, `some`, `even`, `only`, `also`, `more`, `most`, `same`, `than`, `then`, `there`, `here`, `over`, `under`, `before`, `after`, `against`, `between`, `through`, `during`, `without`, `within`.
- Numbers: any token matching `^\d+$`.

**Aggressive list** (anchor only — strips structural and review noise that pollutes title vocabulary):

- All of Standard list.
- **C# keywords / type names:** `public`, `private`, `internal`, `protected`, `static`, `readonly`, `const`, `record`, `class`, `interface`, `struct`, `enum`, `namespace`, `using`, `return`, `void`, `var`, `async`, `await`, `task`, `decimal`, `bool`, `string`, `int`, `long`, `byte`, `char`, `float`, `double`, `object`, `guid`, `ireadonlylist`, `ienumerable`, `list`, `dictionary`.
- **Engage panel-known result-record names** (extend as new tools ship):
  `goalresult`, `campaigngroupresult`, `campaignresult`, `segmentresult`, `getgoalsresult`, `getcampaignsresult`, `getsegmentsresult`, `gettoppagesresult`, `tooldescription`.
- **Engage panel-known entity nouns** (these appear in nearly every finding title; treat as structural):
  `goal`, `goals`, `campaign`, `campaigns`, `campaigngroup`, `segment`, `segments`, `page`, `pages`.
- **Common review-vocabulary verbs and noun phrases** (high-frequency, low-specificity in finding titles):
  `missing`, `missed`, `omits`, `omitted`, `omission`, `drops`, `dropped`, `dropping`, `strips`, `stripped`, `stripping`, `imposes`, `imposed`, `imposing`, `claims`, `claimed`, `silent`, `silently`, `returns`, `narrate`, `narrates`, `narration`, `wire`, `wire-format`, `wireformat`, `tool`, `description`, `descriptions`, `field`, `fields`, `result`, `results`, `record`, `records`, `name`, `names`, `naming`, `rename`, `renamed`, `renames`, `flag`, `flags`, `flagged`, `flagging`, `marketer`, `marketers`, `llm`, `model`, `models`, `engage`, `engageai`.
- **Common review-vocabulary adjectives:** `broken`, `invalid`, `inverted`, `configurable`, `generic`, `imposed`. Wait — `inverted` and `invalid` are content-specific (they're literally what some findings are about). Do NOT stopword these.

**Rule:** if a token is *the literal name of a field the finding is about* (e.g., `isinverted`, `isinvalid`, `monetaryvalue`, `goaltype`), it must remain in the anchor set. Stopwording removes structural prefix words around field-name tokens, never the field-name tokens themselves.

### File-path extraction (unchanged)

For related-Evidence grouping, also extract a normalized file path:
- Match the first occurrence of a path-like token (`.+\.(cs|md|json|csproj|sln|...)`).
- Strip leading `src/`, `D:/source/work/...`, repo-prefix patterns.
- Normalize separators to `/`.

Two findings share `evidence_file` iff their normalized file paths are equal.

### Worked example (Bug §1 fix verification)

Finding 1 (Brand-Voice): `### Finding 1: MonetaryValue rename promotes configurable decimal to currency`

- Anchor raw: monetaryvalue, monetary, rename, promotes, configurable, decimal, currency
- Anchor after aggressive strip (removes `decimal` as C# keyword, nothing else): **{monetaryvalue, monetary, rename, promotes, configurable, currency}**

Finding 2 (Funnel-Stage): `### Finding 1: GoalResult drops IsInverted — LLM will celebrate failure modes as top performers`

- Anchor raw: goalresult, drops, isinverted, celebrate, failure, modes, top, performers
- Anchor after aggressive strip (removes `goalresult`, `drops`, `llm`): **{isinverted, celebrate, failure, modes, top, performers}**

Anchor-token overlap: ∅. **Not related.** ✓ (Bug §1 over-merge fixed.)

Finding 3 (Data-Trust): `### Finding 3: MonetaryValue overclaims the semantic of a free-form decimal`

- Anchor after aggressive strip: **{monetaryvalue, overclaims, semantic, free-form}** (decimal stripped as C# keyword)

Anchor overlap with Finding 1: `{monetaryvalue}` — 1 token. **Related** ✓ (intended same-field merge preserved).

The threshold rule (Step 4a) uses ≥1 anchor token, supplemented by framework-citation check.

---

## §5 — Persona ID extraction

Persona IDs are derived from the dispatched persona file path:

```
personas/01-brand-voice-marketer.md     → brand-voice
personas/02-funnel-stage-marketer.md    → funnel-stage
personas/03-hypothesis-driven-marketer.md → hypothesis-driven
personas/04-data-trust-marketer.md      → data-trust
```

Use these short IDs in all orchestrator-generated output. They appear in:
- `multi_persona_agreement` lists
- `compound_flag` strings
- Run-health rows
- Context-gap routing tables

---

## §6 — Failure detection patterns

Patterns indicating a malformed or refused subagent response (Step 2b retry trigger):

### Refusal patterns

```
^I (cannot|can't|am unable to|won't|will not)
^I apologize, but
^Sorry,? (but )?I (cannot|can't)
^As an AI (assistant|language model)
```

If the response starts with any of these → malformed (skip §1 extraction, go to retry).

### Empty / near-empty response

```
^\s*$               # entirely empty
^\s*[.]{1,10}\s*$   # just dots
```

→ malformed.

### Partial template

If the response contains `### Finding` headers but at least one finding is missing one of the 5 required fields (Severity / Evidence / Framework citation / Issue / Recommendation) → malformed.

If the response contains neither `### Finding` headers NOR a recognized sentinel → malformed.

---

## §7 — Implementation note for Claude (the orchestrator)

When you (Claude, running as the orchestrator) parse persona outputs, you do NOT need to invoke an external regex engine. Read the persona response in-session and apply these patterns mentally — they are the authoritative *shape* definitions, not literal code to execute.

For each persona output, walk the structure:

1. Is the response a sentinel? (§2 — check first; cheapest)
2. Does it contain refusal patterns or empty body? (§6 — bail to malformed)
3. Iterate finding blocks (§1), extracting one structured record per `### Finding N:` header.
4. Scan for Context-gap lines (§3) after the last finding (or after sentinel).
5. For each extracted finding, derive Evidence tokens (§4) for use in Step 4.

If any extraction is ambiguous (rare — only when persona violates its own output template), prefer marking the persona `malformed` and triggering retry over guessing at the intent. False negatives (missing a real finding) are better than false positives (fabricating structure).
