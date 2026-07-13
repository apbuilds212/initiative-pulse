# Extraction Prompt — messy input → validated Initiative Pulse record

**Purpose.** This is the prompt that makes the structured update *cheaper than the
old freeform one.* A PM pastes whatever they already have — a rambling Slack
paragraph, a Jira export, last week's record — and the model returns one clean
record in the schema. The PM spends ~2 minutes confirming, not 20 writing.

**Where it runs — on the stack JKLM already uses.** This isn't a new tool. It
runs in **Claude**, using Claude's **connectors (MCP)** to the tools the org already
lives in:

- **Reads** the initiative's signals from **Jira** (issue/epic + current stage) and
  the working **Slack** channel (this week's thread) — via the Atlassian and Slack
  connectors. The PM doesn't re-type status; Claude pulls it from where it already is.
- **Writes** the validated record into a **Notion database** (the source of truth) —
  via the Notion connector's `create-pages` / `update-page`. Notion *is* the living
  record; the YAML in this repo is the portable, demo-sized representation of one row.

The PM never sees YAML. They run one Claude prompt (or a saved Claude Project / Slack
workflow), eyeball what Claude pulled, and confirm. ~2 minutes.

**Why this beats each tool's native AI.** Notion AI, Jira's Rovo, and Slack AI each
summarize *their own silo*. The initiative isn't one object in any of them — it's
fragmented across all three. This step's job is the cross-tool part: read the three
fragments, reconcile them into one schema-valid record, and flag what it had to infer.

**The guardrail that matters.** The model must NEVER invent a status. If the input
is thin or ambiguous, it fills what it can, leaves the rest empty, and stamps
`confidence: needs_human_confirm` so the record is flagged before it reaches
leadership. (This is the fix for the first failure mode we hit — see the
"What broke" note at the bottom.)

---

## System prompt

```
You convert a Product Manager's raw, unstructured status input into ONE
Initiative Pulse record that strictly conforms to the schema below. You are an
extractor, not an author: you restructure what is present, you do not invent.

SCHEMA (required fields must always be present):
  id              string  slug like "pay-001" — reuse if given, else null
  name            string
  mission_team    string  one of: Payroll, Benefits, Compliance, Platform
  pm_owner        string
  pdlc_stage      enum    Discovery|Definition|Build|QA|GTM|Launched|Paused
  rag             enum    green|amber|red
  what_moved      string  what CHANGED since last update; 1–3 sentences
  blockers        list    each: {issue, owner, since?}; [] if none
  next_milestone  object  {label, date(YYYY-MM-DD)}
  decisions       list    each: {decision, date, by?}; [] if none
  links           object  {spec?, jira?, slack?}
  updated_at      date    today's date (YYYY-MM-DD)
  confidence      enum    confirmed | needs_human_confirm

RULES
1. Restructure only. Do not add facts, blockers, decisions, or dates that are
   not in the input. Empty is better than invented.
2. RAG: choose green/amber/red ONLY if the input gives clear evidence
   (e.g. "blocked", "at risk", "on track", a slipped date). If there is no
   evidence, leave rag as the prior value if provided, else set
   confidence=needs_human_confirm and make your best guess explicit.
3. Set confidence=needs_human_confirm if ANY of:
   - rag had to be guessed without clear evidence,
   - a required field is missing from the input and you had to infer it,
   - the input is internally contradictory.
   Otherwise confidence=confirmed.
4. Dates: normalize to YYYY-MM-DD. Never fabricate a date — if a milestone has
   no date, set date=null and confidence=needs_human_confirm.
5. what_moved is the DELTA, not a description of the project. If you can't tell
   what changed, say so plainly and flag for confirmation.
6. Output VALID YAML only. No commentary, no markdown fences.

After the record, output one line beginning "REVIEW:" listing exactly which
fields you inferred or were unsure about (or "REVIEW: none").
```

## User message (template the PM fills)

```
Raw input:
<<<
{paste Slack message, Jira export, email, or last week's record here}
>>>

If you have last week's record for this initiative, paste it too so I update
rather than start over:
<<<
{optional prior record}
>>>
```

---

## Worked example A — connector input (Jira issue + Slack thread)

This is the *native* path: the PM doesn't type anything. Claude pulls the Jira issue
and the working Slack thread through its connectors, and reconciles both into one
record. Below is the kind of raw payload those connectors return.

**Input (what the Jira + Slack connectors return — Claude reads this, the PM doesn't):**

```
[Jira · CMP-4]  Multi-State Tax Compliance Engine
  status: In Progress   |  assignee: (unassigned)  |  epic: Compliance / Tax 2026
  description: "Current focus: QA validation across all 50 states before launch."
  updated: 2026-06-02
  last comment (QA, 2026-05-27): "Reciprocal NY/NJ withholding returns wrong
  values — failing 6 of the 50-state suite. Blocking sign-off."

[Slack · #initiative-pulse-demo thread, 2026-05-29 → 06-02]
  @ademar: agreed on the call — we hold the launch until the reciprocal calc passes
          full QA. not shipping a wrong withholding number.
  @ademar: also still waiting on data platform for the 2026 state tax tables, can't
          fully revalidate without them.
  @dataplatform-bot: tables ETA pushed to next week.
  @ademar: targeting full 50-state re-validation by 6/12.
```

**Output (schema-valid — written to the Notion DB via the connector):**

```yaml
id: cmp-003
name: Multi-State Tax Compliance Engine
mission_team: Compliance
pm_owner: Ademar Perez                      # from Slack (Jira issue is unassigned)
pdlc_stage: QA                              # from the Jira description ("current focus: QA validation"); Jira status is "In Progress"
rag: red
what_moved: >
  QA found incorrect withholding for NY/NJ reciprocal agreements (failing 6 of 50
  states); launch held until it passes. Still awaiting 2026 tax tables (ETA slipped).
blockers:
  - issue: Reciprocal-state (NY/NJ) withholding calc returns wrong values
    owner: Ademar Perez
    since: "2026-05-27"
  - issue: Awaiting updated 2026 state tax tables
    owner: Data Platform
    since: "2026-05-29"
next_milestone:
  label: Re-validate withholding across all 50 states
  date: "2026-06-12"
decisions:
  - decision: Hold launch until reciprocal-state calc passes full QA
    date: "2026-05-29"
    by: Ademar Perez
links:
  jira: https://astorpoint.atlassian.net/browse/CMP-4
  slack: https://astorpoint.slack.com/archives/C0BB5J5BKS5
updated_at: "2026-06-02"
confidence: confirmed
REVIEW: pdlc_stage=QA derived from the Jira description ("current focus: QA validation"),
since the Jira status itself is "In Progress"; pm_owner taken from Slack as the issue is
unassigned in Jira; rag=red derived from the QA "blocking sign-off" comment + the explicit
launch hold (clear evidence, not inferred).
```

**The point:** every field traces back to a Jira field or a Slack message — Claude
*reconciled two tools into one record*, which is the exact thing no single tool's AI
does. The PM confirms in seconds; the record lands in Notion; the dashboard and rollup
read it from there.

---

## Worked example B — freeform paste (the fallback path)

Not every initiative has clean Jira/Slack signal. When a PM would rather just paste a
paragraph (or there's no connector for a given source), the same prompt handles it.

**Input (messy, the kind PMs actually send):**

```
multi state tax thing — qa found the ny/nj reciprocal withholding is calculating
wrong, also still waiting on data platform for the 2026 tax tables. ademar owns it.
we agreed last fri to hold the launch til it passes full qa. trying to revalidate
all 50 states by ~6/12. jira CMP-4
```

**Output (schema-valid):**

```yaml
id: cmp-003
name: Multi-State Tax Compliance Engine
mission_team: Compliance
pm_owner: Ademar Perez
pdlc_stage: QA
rag: red
what_moved: >
  QA found incorrect withholding calculation for NY/NJ reciprocal agreements;
  launch is on hold. Awaiting 2026 state tax tables from data platform.
blockers:
  - issue: Reciprocal-state (NY/NJ) withholding calc returns wrong values
    owner: Ademar Perez
    since: "2026-05-27"
  - issue: Awaiting updated 2026 state tax tables
    owner: Data Platform
next_milestone:
  label: Re-validate withholding across all 50 states
  date: "2026-06-12"
decisions:
  - decision: Hold launch until reciprocal-state calc passes full QA
    date: "2026-05-29"
    by: Ademar Perez
links:
  jira: https://astorpoint.atlassian.net/browse/CMP-4
updated_at: "2026-06-02"
confidence: confirmed
REVIEW: inferred pm_owner full name ("ademar" → "Ademar Perez"); inferred decision date from "last fri".
```

Note the model surfaced its two inferences in the REVIEW line instead of burying
them — so the PM confirms in seconds and the record is trustworthy by the time
leadership reads it.

---

## What broke — and the fix (the thin-input test)

This is the honest failure mode we hit, preserved as the second worked example.
The input below is the real kind of non-update PMs send when nothing has moved:

**Input (thin — this is all we got):**

```
notifications platform — sam's still heads down on the audit stuff, nothing
really new this week. think we're fine
```

**v1 output (the broken prompt) — looks plausible, mostly invented:**

```yaml
id: plt-005
name: Unified Notifications Platform
mission_team: Platform
pm_owner: Sam Ortiz
pdlc_stage: Discovery
rag: green                                  # INVENTED — "think we're fine" is not evidence
what_moved: >
  Notification-path audit progressing well; consolidation
  proposal on track for review.             # INVENTED — input says nothing moved
blockers: []                                # WRONG — dropped last week's open scope blocker
next_milestone:
  label: Complete audit and circulate proposal
  date: "2026-06-13"                        # FABRICATED — no date anywhere in the input
updated_at: "2026-06-02"
confidence: confirmed                       # the worst part: it reported itself as trustworthy
```

Schema-valid, confidently wrong. If this had shipped, leadership would have read
a green status that no human ever stated. That's *worse* than the old manual
process — same bad data, more trust.

**v2 output (current prompt, rules 1–5 applied):**

```yaml
id: plt-005
name: Unified Notifications Platform
mission_team: Platform
pm_owner: Sam Ortiz
pdlc_stage: Discovery
rag: amber              # carried from prior record — input gave no evidence to change it
what_moved: >
  No substantive change reported this week; PM indicates audit
  work is still in progress.
blockers:
  - issue: No agreement on whether to migrate legacy email paths in v1
    owner: Sam Ortiz
    since: "2026-05-18"   # carried from prior record — not resolved in input
next_milestone:
  label: Reach scope decision on legacy migration
  date: "2026-06-20"      # carried from prior record
updated_at: "2026-06-02"
confidence: needs_human_confirm
REVIEW: rag carried from prior record ("think we're fine" is not status evidence);
blockers and milestone carried forward unverified; what_moved is effectively "no update" —
recommend PM confirms blocker is still open.
```
