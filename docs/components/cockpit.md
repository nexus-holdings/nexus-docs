# Cockpit

<p class="lede">Cockpit was the standalone Next.js dashboard for the Nexus platform — chairman-facing UI for company oversight, ticket flow, and live transcript viewing. <strong>It's archived</strong> — no longer actively developed, no systemd unit, no auto-start — though the old checkout is sometimes left running locally on <code>:3000</code> when an operator wants the old UI. A replacement is being built as a plugin rendered inside the governance company.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> archived</span>
  <span>Updated 2026-05-19</span>
  <span>Owner: Platform</span>
</div>

!!! warning "Archived — pending rewrite"
    The original Cockpit is no longer the canonical UI. As of 2026-04-18 the company is marked `status=archived` in the Paperclip API and there's no systemd unit driving it. You may still see a local Next.js process listening on `:3000` — that's an operator running the old checkout ad-hoc, not a maintained service. The replacement is in progress as a plugin inside the governance company. This page describes what the original Cockpit was (for historical context) and where to look for its successor when it lands.

## What it was

A Next.js 16 / React 19 / Tailwind dashboard running at `http://localhost:3000`, reading from Paperclip's HTTP API and the metrics PostgreSQL database. Single-page operator surface for the substrate.

| Property | Original value | Current state |
|---|---|---|
| **Path** | `~/Projects/nexus/cockpit/` | Still on disk; not actively developed |
| **Stack** | Next.js 16, React 19, Tailwind, Recharts | Frozen at last working commit |
| **Port** | `3000` | Only runs if explicitly started |
| **systemd unit** | _(never had one)_ | Not present in system or user systemd — launched ad-hoc when needed |
| **Status in Paperclip** | (was) `active` | `archived` |

## What it surfaced

The dashboard had four main sections (preserved here so the replacement has a feature target):

| Section | Purpose |
|---|---|
| **Metrics tab** | Portfolio-level dashboards — issue throughput, agent performance, eval scores, infra health |
| **Active companies** | One row per active company with ticket counts, recent activity, current dispatches |
| **Ideas & conversations** | Strategic prompts from the Chairman; tracking decisions in flight |
| **Live transcript** | Scrolling session output for the currently-dispatched agent (WebSocket-backed) |

The "Live transcript" feature was the operationally most-used. Its absence is what makes the Cockpit deprecation visible — users currently fall back to `tail -f` on the JSONL session log.

## Why it was archived

Two reasons converged:

1. **Architectural fit** — keeping a standalone dashboard alongside an autonomous-AI-company platform created a parallel UI track that the rest of the substrate didn't speak to. The plugin model (where Paperclip surfaces UI via the Plugins overlay) gives a tighter coupling: the dashboard becomes part of the same system it monitors.

2. **Maintenance cost vs. usage** — Cockpit had its own dependency tree (Node 22, ~600 npm packages), its own dev server, its own deploy story. For a single-operator tool, that overhead didn't pay for itself.

The decision is recorded in the `project_cockpit_rewrite.md` memory note (2026-04-18).

## Where the replacement is going

The replacement is being built as a **plugin inside the governance company** — meaning the operator UI lives at the same surface as Paperclip's plugin slots, with the same auth, the same data model, no separate stack to keep alive.

Operationally:

- Read endpoints come from [Paperclip](paperclip.md)'s API + [Nexus MCP](nexus-mcp.md)'s compound tools (e.g., `nexus_status`, `performance_report`)
- Live session view tails the same JSONL transcripts that [Nexus Memory](nexus-memory.md) ingests
- No new database, no new processes — leans on what's already there

The plugin is not yet shipped (as of this writing). Status will move from "pending rewrite" to "active" when it lands.

## If you want the old Cockpit anyway

For local development against a working dashboard, the archived checkout still runs:

```bash
cd ~/Projects/nexus/cockpit
npm install                       # if first time on a fresh checkout
npm run build && npm run start    # production-mode is more stable than dev
# Browser: http://localhost:3000
```

But: no fixes are being landed against it. If something breaks, it stays broken. The recommended path for live operations is `nexus_status` from [Nexus MCP](nexus-mcp.md) plus direct queries against [Paperclip](paperclip.md)'s API.

## See also

- [Paperclip](paperclip.md) — the API backend Cockpit read from
- [Nexus MCP](nexus-mcp.md) — the chairman-level tools that effectively replace much of what Cockpit surfaced
- [Nexus Memory](nexus-memory.md) — where live transcripts live (the JSONL that the Live transcript tab tailed)
- [Heartbeat](../concepts/heartbeat.md) — the dispatch loop Cockpit visualised
