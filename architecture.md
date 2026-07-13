# Architecture — Initiative Pulse on the JKLM stack

**The one-line version:** Initiative Pulse is not a new tool. It's a thin structured
layer plus **Claude as the orchestration engine**, riding on the four tools JKLM
already uses — Claude, Notion, Jira, and Slack.

## The problem, stated in stack terms

JKLM has four AI-capable tools. Each has strong AI *inside its own walls*:

| Tool | Its native AI does… | …but only over |
|---|---|---|
| **Jira** (Rovo) | summarize issues, agentic work items | Jira |
| **Slack** | channel/thread summaries, AI workflows | Slack |
| **Notion** | Custom Agents, Database Agents (3,000+ records) | Notion |
| **Claude** | reason + read/write across all of them via connectors | *the whole stack* |

The catch: **an initiative is not a single object in any of them.** Its status lives in
a Jira epic, its decisions get socialized in a Slack thread, its spec sits in Notion.
So each tool's AI can only ever see a slice. No tool produces a cross-tool,
portfolio-level, structured record — because that requires (a) a shared schema and
(b) something that reconciles the three tools into it. That "something" is Claude.

## The flow

```
        SOURCES (where work already happens)                 ENGINE                  SOURCE OF TRUTH            CONSUMERS
        ─────────────────────────────────────                ──────                  ───────────────           ─────────

   ┌─────────────┐  Jira issue + stage   ┐
   │   JIRA      │ ───────────────────►  │
   │ (Rovo MCP)  │   CMP-4 "In Progress" │
   └─────────────┘                       │        ┌────────────────────┐        ┌──────────────────┐      ┌──────────────────────┐
                                          ├──────► │      CLAUDE        │ ─────► │   NOTION DB      │ ───► │  Dashboard / Notion  │
   ┌─────────────┐  working thread       │        │  extraction prompt │ write  │  one row per     │ read │  view (leadership     │
   │   SLACK     │ ───────────────────►  │        │  reconciles the    │ (MCP)  │  initiative =    │      │  self-serves)        │
   │ (Slack MCP) │  #compliance-tax      │        │  fragments into    │        │  the schema      │      └──────────────────────┘
   └─────────────┘                       ┘        │  ONE record        │        │  contract        │      ┌──────────────────────┐
                                                   └─────────┬──────────┘        └────────┬─────────┘ ───► │  Weekly rollup        │
   ┌─────────────┐  freeform paste                          │ flags inferred             │ read           │  posted to SLACK      │
   │  PM (opt.)  │ ──────────────────────────────────────►  │ → needs_human_confirm      │                │  (scheduled Claude    │
   └─────────────┘                                          ▼                            │                │   task / Notion agent)│
                                                   PM confirms (~2 min)                   │                └──────────────────────┘
                                                                                          ▼
                                              freshness / coverage / RAG counts computed FROM the record — no manual audit
```

## Step by step

1. **Read (Jira + Slack connectors).** Claude pulls the initiative's Jira issue
   (status, assignee, last QA comment) and its working Slack thread. The PM re-types
   nothing. A freeform paste is also accepted when there's no clean signal.

2. **Reconcile (extraction prompt, in Claude).** Claude restructures the two fragments
   into one schema-valid record. It never invents: anything it had to infer is stamped
   `confidence: needs_human_confirm` and listed in a `REVIEW:` line. *(See the preserved
   v1→v2 failure story in `extraction-prompt.md` — this guardrail is the lesson learned.)*

3. **Write (Notion connector).** The validated record is written to the **Notion
   database** (`create-pages` / `update-page`) — one row per initiative. Notion is the
   living source of truth; `initiatives.yaml` / `status-schema.yaml` in this repo are the
   portable, demo-sized representation of that DB.

4. **Consume — without anyone being pinged:**
   - **Rollup (scheduled Claude task / Notion Custom Agent):** reads all rows, generates
     the weekly leadership summary, and **posts it to Slack**. Replaces the 2–3 hr compile.
   - **Dashboard / Notion view:** reads the same rows and renders the portfolio board.
     Freshness, coverage, and the AI-inferred count are computed live — the system
     measures its own adoption with no manual audit.

## Why Claude, specifically

Claude is the only layer that spans all four tools (read **and** write via MCP
connectors, set up in under a minute, respecting each user's existing permissions). The
extraction and rollup steps *are* Claude prompts; the connectors are how it reaches the
data. That makes Claude the connective tissue that turns three silos into one system —
which is exactly the role's mandate: *use AI to redesign the internal systems that power
Product.*

## What's demo vs. production

| | Demo (live + offline fallback) | Production (their stack) |
|---|---|---|
| Source of truth | Notion DB (live) — `initiatives.yaml` as offline fallback | Notion database |
| Inputs | Live Jira (CMP-4) + Slack (#initiative-pulse-demo) via connectors — connector-shaped sample as fallback | live Jira + Slack via connectors |
| Rollup target | Posted to `#initiative-pulse-demo` (live) — markdown output as fallback | posted to a Slack channel |
| Dashboard | `dashboard.html` (self-contained) reflects the same records | a Notion DB view / embedded board |

The live connectors are the primary demo path. The offline samples are the instant fallback
if anything stalls — the architecture is identical either way.

**This is already real in Notion.** A live Initiative Pulse database exists in the connected
Notion workspace, seeded with all 7 records via the Notion connector:
https://app.notion.com/p/23e2491f018a455f96f6067c05399835 — Claude built the schema and wrote
every record through MCP, exactly as described above.

**Slack is also live.** A private channel `#initiative-pulse-demo` in the connected workspace
holds a real status thread (Claude reads it) and a posted weekly leadership rollup (Claude wrote
it) — the read and write sides of the Slack edge, demonstrated end-to-end.

**Jira is now live too.** A real issue **CMP-4** ("Multi-State Tax Compliance Engine," *In Progress*,
with the NY/NJ reciprocal-withholding QA comment) exists at astorpoint.atlassian.net/browse/CMP-4.
So the full cross-tool flow is real: Claude reads **Jira (CMP-4)** + **Slack** via connectors,
reconciles them into one record, writes it to **Notion**, and posts the rollup back to **Slack** —
every edge in the diagram above is a live connector call, not a mock.
