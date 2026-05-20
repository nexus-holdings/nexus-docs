# Tickets

<p class="lede">A single, agent-executable unit of work. The ticket lifecycle is Nexus's most important state machine — every other concept (heartbeat, agents, postmortems) hangs off it.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-19</span>
  <span>Owner: Platform</span>
</div>

## The states

| Status | Meaning |
|---|---|
| `backlog` | Filed but not yet scheduled. No agent is working on it. |
| `todo` | Queued for dispatch. Next in the company's work queue. |
| `in_progress` | An implementer agent is actively working on it. |
| `in_review` | Implementation complete, code pushed to a feature branch, reviewer is evaluating. |
| `done` | Reviewed **and merged to `main`**. The terminal success state. |
| `cancelled` | Explicitly killed. Terminal, non-success. |
| `blocked` | Agent hit an unresolvable blocker. Heartbeat re-tries after manual intervention or escalation. |

## The flow

<img src="../../assets/diagrams/tickets-flow.svg" alt="Ticket lifecycle state diagram — backlog → todo → in_progress → in_review → done along the top (forward solid arrows), with blocked as an off-path state below in_progress. Dotted backward arrows show unblock (blocked → todo) and reject (in_review → todo) routed through margin space to avoid the happy path.">

## What `done` guarantees

A ticket in `done` status **has its feature branch merged to `main`**. Full stop.

If the feature branch is not on main, the ticket is not done — it may be `in_review` (verdict pending), or it may have drifted (bug). Two important corollaries:

- `done ≠ reviewed`
- `done = reviewed AND merged`

## Transition responsibilities

### `backlog → todo`

The heartbeat (or a manual board action) promotes the highest-priority `backlog` ticket to `todo` when there's session-cap headroom.

### `todo → in_progress`

Dispatch: the heartbeat spawns an implementer agent and transitions the ticket. The agent works on a branch named `ticket/<shortId>`.

### `in_progress → in_review`

Implementer-driven. When the agent finishes, it:

1. Pushes the branch
2. Posts a completion comment with branch + SHA
3. Exits

The session-complete handler transitions the ticket to `in_review` on success, or back to `in_progress` on failure (for retry).

### `in_review → done` (or rejected, or escalated)

Reviewer-driven. The reviewer agent's decision is binary in code (`approved: true|false`), but in practice there are three operational paths:

| Verdict | Effect |
|---|---|
| **Approve** | Triggers the merge-agent. On successful merge → `done`. |
| **Reject** | Ticket goes back to `todo`. Implementer retries with reviewer feedback. |
| **Escalate** | Manual path: reviewer leaves the ticket in `in_review` and flags it for a human. Not a first-class code construct — by convention only. |

### `blocked`

An implementer can mark a ticket `blocked` when it hits an unresolvable obstacle (missing context, broken dependency, ambiguous requirement). The heartbeat skips `blocked` tickets until a human or an escalation flow flips them back to `todo`.

## Ticket fields

| Field | Purpose |
|---|---|
| `id` | Short ID (e.g., `NEXA-58`). Human-readable, stable. |
| `uuid` | Internal identity. Used in API calls. |
| `title` | One-line summary. |
| `description` | Full body. Should include acceptance criteria. |
| `company_id` | Which company owns this ticket. |
| `priority` | `low` / `medium` / `high` / `critical`. Drives backlog ordering. |
| `labels` | Tags. Some labels gate agent selection (e.g., `mechanical` → cheaper model). |
| `status` | Current state (see above). |
| `assignee` | Agent identity (auto-set on dispatch). |

## Why this matters

The ticket model is what lets Nexus stay **observable**. Every ticket has:

- A clear acceptance criterion (or it's not a ticket)
- An immutable lifecycle (status transitions are recorded)
- A linked session transcript (in MemPalace once the agent finishes)
- A linked branch + commits (for code work)
- A postmortem if anything went sideways

That observability is what makes the substrate compound across hundreds of agent runs.

## See also

- [Heartbeat](heartbeat.md) — what picks tickets up
- [Companies](companies.md) — where tickets live
- [Sessions vs. Tickets](sessions-vs-tickets.md) — why these are two different units
- [Paperclip](../components/paperclip.md) — the orchestrator that stores ticket state
- [Nexus Core](../components/nexus-core.md) — the runtime that dispatches them
- [Debug a Ticket](../guides/debug-a-ticket.md) — operational guide
