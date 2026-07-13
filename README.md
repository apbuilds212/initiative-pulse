# Initiative Pulse

A structured, self-serve source of truth for Product initiatives — so status is
*read*, not *re-collected*, and PMs spend time on product, not coordination.

**It's not a new tool.** It runs on the stack JKLM already uses — **Claude**
as the engine, reading **Jira** and **Slack**, writing the record into **Notion**,
posting the rollup back to **Slack**. No new place to log in.

**Live build — all three tools wired for real:**

- **Notion** — a working Initiative Pulse database, seeded with the full sample portfolio plus
  a "Leadership View" board grouped by RAG: [open it here](https://app.notion.com/p/23e2491f018a455f96f6067c05399835).
  Claude built and populated it through the connector; the same prompt writes new records into it.
  A deployable dashboard (`initiative-pulse-live.html` + `initiative-pulse-final.zip`) reads this
  database live via a Netlify serverless proxy — see `netlify-setup.md`.
- **Slack** — a private `#initiative-pulse-demo` channel with a real status thread Claude reads
  and a posted weekly leadership rollup Claude wrote. The full read→write loop, live.
- **Jira** — a real issue **CMP-4** ("Multi-State Tax Compliance Engine," *In Progress*, with the
  NY/NJ reciprocal-withholding QA comment) at astorpoint.atlassian.net/browse/CMP-4. Claude reads
  it live via the connector.

Claude reads **Jira (CMP-4) + Slack**, reconciles them, writes the record to **Notion**, and posts
the rollup to **Slack** — the whole flow runs on real tools.

## The problem this fixes

The PDLC standardized the *process* (Jira statuses, a spec template, a kickoff
format) but not the *data*. Status lives in 6–8 people's heads, decisions get
socialized across "four or five conversations," and every downstream artifact —
the weekly leadership summary, the kickoff re-walk, QA scenarios, GTM briefs — has
to be **re-derived by a human every time**. The weekly status compile alone costs
one person 2–3 hours and produces inconsistent inputs.

**Root cause:** the org has four AI-capable tools (Claude, Notion, Jira, Slack),
but an initiative isn't a single object in any of them — it's fragmented across
Jira, Slack, and Notion, so each tool's AI can only ever see its own slice. There
is no shared, structured, living record that spans them. Initiative Pulse is that
shared record, and Claude is the glue that keeps it current from the other three.

**"Why not just use Notion AI / Rovo / Slack AI?"** Because each of those
summarizes *its own silo*. None produces a cross-tool, portfolio-level structured
record — that requires a shared schema and something reconciling the tools into it.
That's exactly what this is.

## What's in this folder

| File | What it is |
|---|---|
| `architecture.md` | How it rides on the existing stack: Claude reads Jira + Slack → writes the record to Notion → posts the rollup to Slack. The connector map. Start here. |
| `status-schema.yaml` | The structured record definition + a JSON Schema for validation. This is the contract — and it maps 1:1 to a Notion database. |
| `initiatives.yaml` | The portfolio — one record per initiative (sample JKLM data). The portable, demo-sized export of the Notion DB. |
| `extraction-prompt.md` | Claude prompt: reads Jira issue + Slack thread (via connectors) → one validated record written to Notion. Worked example A shows the connector path; B shows the freeform fallback. |
| `rollup-prompt.md` | Claude prompt (scheduled task / Notion agent): all records → the weekly leadership summary, posted to Slack automatically. Replaces the 2–3 hr manual compile. |
| `initiative-pulse-live.html` | The live portfolio board (the Notion DB's leadership view) anyone opens anytime. Reads the Notion database directly through a serverless proxy and computes freshness & coverage from the data — the "surface without a manual audit" proof. |
| `initiative-pulse-final.zip` | The deployable Netlify bundle: `index.html`, `logo.webp`, `netlify.toml`, and the `notion-proxy.js` function that keeps the Notion token server-side. Drag-and-drop deploy. |
| `netlify-setup.md` | Three-step guide to deploy the live dashboard to Netlify so it reads directly from Notion (create integration → set `NOTION_TOKEN` → deploy the zip). |
| `Initiative-Pulse-Onsite-Deck.pptx` | The onsite presentation deck walking through the problem, the stack-native solution, and the live demo. |

## How a PM uses it Monday morning (≈2 minutes)

1. Run the **extraction prompt** in Claude. It pulls this initiative's Jira issue
   and Slack thread through the connectors — you don't re-type anything. (Or paste
   a freeform update if you'd rather; the prompt handles both.)
2. Claude returns a clean record, writes it to the **Notion** database, and flags
   anything it inferred (`needs_human_confirm`). You eyeball it, fix one thing, confirm.
3. That's it. You're done. **No one will Slack you for a status this week.**

Meanwhile, automatically:

- The **rollup prompt** runs over all records (e.g. Monday 7am as a scheduled
  Claude task / Notion agent) and **posts the leadership summary to Slack** — no one
  compiles it by hand.
- The **dashboard / Notion view** always reflects the latest records. Leadership
  self-serves instead of asking. Kickoffs open with the current record instead of a
  20-minute re-walk of a stale spec.

## Open the live dashboard

`initiative-pulse-live.html` is the leadership board. The KPI strip (freshness %,
complete-records %, RAG counts, AI-inferred count) is computed live from the
records, and you can filter by RAG or team.

To run it against the actual source of truth, deploy `initiative-pulse-final.zip`
to Netlify (follow `netlify-setup.md`): it reads the Notion database directly
through a serverless proxy, so the dashboard always reflects the latest records
with no manual export. The proxy keeps your `NOTION_TOKEN` server-side — it never
reaches the browser. `initiatives.yaml` remains the portable, demo-sized stand-in
for the Notion DB.

## The guardrail (and the honest failure mode)

The first version of the extraction prompt over-trusted thin input and *invented*
a RAG status. The fix: the prompt now restructures only what's present, leaves
gaps empty, and stamps `confidence: needs_human_confirm` on anything it inferred —
so unverified records are visibly flagged on the dashboard and in the rollup
footer before they ever reach leadership. AI drafts; a human owns the truth.

## How it measures its own success (no manual audit)

- **Hours reclaimed** — the 2–3 hr weekly manual compile goes to zero; PM update
  time target <5 min/initiative.
- **Freshness** — % of initiatives updated in the last 7 days (dashboard computes it).
- **Coverage** — % of active initiatives with a complete structured record.
- **Leadership pull-down** — dashboard opens vs. "where's the status?" pings → trends to zero.
- **Adoption** — # of teams self-serving updates without being chased.

Every number is read off the data the system already holds.

## Where this goes next

Status is the entry point. The *same* structured record is the missing input for the
other three symptoms — so the roadmap is: **status → spec hygiene → QA scenario
generation → GTM briefs**, each generated from the record instead of rebuilt by
hand. One source of truth, many artifacts, zero re-keying.

---
*Built by Ademar Perez for the JKLM Product Operations Manager assessment.*
