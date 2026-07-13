# Rollup Prompt — all records → weekly leadership summary

**Purpose.** This replaces the 2–3 hours someone spends every week Slacking 6–8
leads and hand-formatting a summary. It takes the full set of structured records
and emits the leadership update automatically. Because it reads structured data,
the output is consistent every week — no more "inconsistent inputs."

**Where it runs — on the existing stack.** As a **scheduled Claude task** (e.g. Monday
8am) — or, equivalently, a **Notion Custom Agent** on a schedule. It reads all records
from the **Notion database** (the source of truth) and **posts the summary to the
leadership Slack channel** via the Slack connector. Or on demand. Nobody gets pinged for
an update, and no one compiles anything by hand. `initiatives.yaml` is the demo-sized
stand-in for the Notion DB it reads in production.

---

## System prompt

```
You write the weekly Product portfolio update for leadership from a set of
structured Initiative Pulse records (YAML). You summarize the data as-is. You do
not invent status, and you do not soften red items.

INPUT: a list of initiative records conforming to the Initiative Pulse schema.

PRODUCE, in this order:

1. HEADLINE (2–3 sentences): portfolio health in plain language. Lead with what
   needs leadership attention. State counts: X green / Y amber / Z red.

2. NEEDS ATTENTION (only red + amber-with-aging-blockers):
   For each: **Name** (team, stage) — the blocker, who owns it, how long open,
   and the next milestone + date. This is the section leadership actually acts on.

3. ON TRACK (green): one line each — name, what moved, next milestone + date.

4. SHIPPED THIS PERIOD (stage = Launched and updated within the period): one line each.

5. DATA QUALITY FOOTER (computed, not prose):
   - Freshness: N of M initiatives updated in the last 7 days (list any stale ones).
   - Needs confirm: list any record with confidence = needs_human_confirm — these
     are AI-inferred and unverified; leadership should treat them as provisional.

RULES
- Order NEEDS ATTENTION by severity: red first, then amber, oldest blocker first.
- Never upgrade a RAG. Report red as red.
- Quote dates as given. If next_milestone.date is null, write "date TBD" and list
  it under data quality.
- Be terse. Leadership reads this in 90 seconds. No filler, no preamble.
- Markdown output, ready to paste into Slack/email.
```

## User message

```
Here are this week's initiative records:
<<<
{fetch from Notion DB: https://app.notion.com/p/23e2491f018a455f96f6067c05399835?v=b7687ce82a2e4eeb8b78d500f4644d6d}
>>>

Reporting period: {e.g. week of 2026-06-02}
```

> **Step 1 (production):** Read initiative records directly from the Notion database at
> https://app.notion.com/p/23e2491f018a455f96f6067c05399835?v=b7687ce82a2e4eeb8b78d500f4644d6d
> using the Notion connector. `initiatives.yaml` remains the demo-sized stand-in for local testing.

---

## Worked example output (run against the 7 sample records)

```markdown
# Product Portfolio — Week of June 2, 2026

**3 green · 3 amber · 1 red.** Most of the portfolio is on track, but the
Multi-State Tax Compliance Engine is RED and holding its own launch — it needs
attention this week. One record (Unified Notifications) is stale and AI-inferred;
treat as provisional.

## ⚠️ Needs attention

- **Multi-State Tax Compliance Engine** (Compliance · QA) — 🔴
  Reciprocal-state (NY/NJ) withholding calc is wrong; launch held until it passes
  full QA. Also awaiting 2026 tax tables (owner: Data Platform, open 4 days).
  Owner: Abel Pena. **Next: re-validate all 50 states — June 12.**
- **Benefits Open Enrollment 2027 Readiness** (Benefits · Definition) — 🟡
  3 of 5 carriers haven't confirmed EDI feed format (open 13 days, owner: Marcus Lee).
  **Next: lock integration scope — June 16.**
- **I-9 / E-Verify Workflow Redesign** (Compliance · Build) — 🟡
  E-Verify vendor sandbox intermittently down, slowing integration (open 7 days,
  owner: Sam Ortiz). **Next: integration code-complete — June 18.**

## ✅ On track

- **Off-Cycle Payroll Self-Serve** (Payroll · Build) — approval-rules engine shipped
  behind a flag; 3 companies ran off-cycle end-to-end. Next: 10-company beta, June 19.
- **Contractor Payments International** (Payroll · GTM) — feature-complete and QA-passed
  for 5 countries; drafting GTM brief. Next: public launch, June 10.

## 🚀 Shipped this period

- **HSA/FSA Contribution Limit Automation** (Benefits) — live to 100% of eligible
  companies, zero support tickets in week one.

---
*Data quality: 6 of 7 initiatives updated in last 7 days. Stale: Unified
Notifications Platform (last update May 21). Needs human confirm: Unified
Notifications Platform (AI-inferred from a thin Slack thread).*
```

The footer is the quiet proof point: the summary tells leadership how trustworthy
it is, computed from the records — no one audited anything by hand.
