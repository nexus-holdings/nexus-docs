# Sessions vs Tickets

<p class="lede">Two distinct primitives that get confused constantly: a <strong>ticket</strong> is a <em>unit of work</em>, a <strong>session</strong> is an <em>agent run</em>. One ticket can outlive many sessions; one session can touch many tickets. The substrate treats them as separate state machines for good reasons.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-19</span>
  <span>Owner: Platform</span>
</div>

## The distinction in one line

> A ticket is **what needs to be done**. A session is **a specific agent's specific attempt to do something**.

The temptation is to collapse the two — "the agent is working on ticket X" sounds like the agent and the ticket are the same object. They aren't, and the substrate keeps them separate because the *failure modes* are different.

## The two state machines

<img src="../../assets/diagrams/sessions-vs-tickets-dual.svg" alt="Two side-by-side state machines: Ticket (DFA 1) on top — backlog → todo → in_progress → in_review → done, with blocked off-path below in_progress and a reject arrow returning from in_review to todo above. Session (DFA 4) below — spawning → idle → processing → closing → closed, with a turn_complete loop returning processing to idle. Accept states (done, closed) marked brass.">


| | Ticket (DFA 1) | Session (DFA 4) |
|---|---|---|
| **What it is** | A unit of work | A subprocess running an agent |
| **Owned by** | A company | The ACP plugin |
| **Lives in** | Paperclip's Postgres | Process memory + ACP plugin state |
| **Identity** | UUID + ticket identifier (e.g. `ENG-42`) | UUID assigned at spawn |
| **Terminal state** | `done` (or `cancelled`) | `closed` |
| **Lifetime** | Days to weeks | Minutes to ~8 hours (max-age cap) |
| **Survives restart** | Yes | No — sessions die with the host |
| **Created by** | A human, agent, or routine | A dispatch event (heartbeat or chat-plugin trigger) |

## How they relate

```mermaid
flowchart LR
    T[Ticket TODO]
    H[Heartbeat picks up]
    S1[Session 1<br/><i>first attempt</i>]
    R1[Review reject]
    T2[Ticket TODO again]
    S2[Session 2<br/><i>retry</i>]
    R2[Review pass]
    D[Ticket DONE]

    T --> H
    H --> S1
    S1 --> R1
    R1 --> T2
    T2 --> S2
    S2 --> R2
    R2 --> D

    classDef brass fill:#14171A,stroke:#D4A574,color:#E6E3DC,stroke-width:1.5px
    classDef other fill:#0D0F11,stroke:#6F7177,color:#E6E3DC,stroke-width:1px
    class T,D brass
    class H,S1,R1,T2,S2,R2 other
```

The ticket outlives both sessions. The sessions terminate; the ticket stays as a queryable record of "this thing still needs doing." When the second session finally produces an approved result, the ticket transitions to `done`.

Inversely, a single session can touch multiple tickets if the agent is asked to bulk-update (e.g. "close all tickets older than 30 days"). The session has one identity; the work it does spans many tickets.

## Why keep them separate?

Three concrete reasons:

### 1. Tickets persist; sessions don't

Restart Paperclip. Tickets are still there (they're rows in Postgres). Sessions aren't — `closed` is terminal for a session, but no one expects a closed session to be re-runnable. The next attempt at a ticket spawns a *new* session.

If the substrate treated session and ticket as one object, every restart would either lose work-in-progress (bad) or invent fake-but-persistent session state (worse).

### 2. Review verdicts attach to *sessions*, not tickets

The review DFA (DFA 2) produces a verdict — `approve`, `reject`, `escalate` — about *what a specific agent run produced*. If a ticket goes through three sessions before being approved, you have three review verdicts, each pinned to its own session.

This matters operationally:
- The merge agent (DFA 3) needs to know which session's branch to merge — there might be three branches across the three sessions
- Performance metrics need to attribute success/failure to the specific *agent + model + session config*, not the ticket
- Postmortems (DFA 20) reference the session that failed, not the ticket — a ticket that eventually succeeds can have postmortems from its earlier failed sessions

### 3. The granularity of "where am I" is different

"What's the ticket status?" is a *workflow* question — answered by querying Paperclip.

"What's the agent doing right now?" is a *runtime* question — answered by tailing the session's NDJSON transcript via the [ACP plugin](../components/plugins/acp.md).

Collapsing the two would force operators to choose between the two views. Keeping them separate means both surfaces stay legible.

## The session ID lives on the ticket

Tickets do track which session is currently in flight against them, but as a *pointer*, not by absorbing the session's state:

```
ticket.in_flight_session_id  →  session UUID  (or null)
```

If the in-flight session terminates without producing a verdict (idle timeout, crashed, killed), the heartbeat detects the inconsistency on its next tick and resets the ticket back to `todo`. The session is gone; the ticket survives. This is the "zombie detection" path in DFA 1.

## When does a session exist *without* a ticket?

Chat-driven sessions, mostly. When a user does `/acp spawn` in a Telegram thread, the ACP plugin spawns a session bound to the *thread*, not to a Paperclip ticket. There's no ticket entity, no Paperclip state. The session lives, does work, and closes on idle or `/acp close`.

This is the asymmetry — every ticket-driven session has a parent ticket, but not every session has one. The substrate accommodates both.

## When are tickets *without* sessions?

Most of their lifetime. A ticket in `backlog` or `todo` has no in-flight session. Tickets in `in_review` have no session either — the reviewer's session has already closed, and the verdict is stored on the ticket. `blocked` tickets are inert until manually unblocked.

In a healthy substrate, the ratio is many tickets : few live sessions. The session budget is small (default 20 across the org); the ticket queue can be hundreds deep.

## How transitions interlock

The two state machines run independently but a few specific events couple them:

| Ticket event | Session event | When |
|---|---|---|
| `assign` (`todo → in_progress`) | `spawn_request` (`init → spawning`) | Heartbeat picks up ticket and dispatches an agent |
| `complete` (`in_progress → in_review`) | `exit_0` (`processing → closed`) | Agent exits successfully and posts a verdict |
| `block` (`in_progress → blocked`) | `exit_nonzero` or `kill` | Agent hits an unresolvable blocker |
| `reset` (zombie detection) | (session already closed) | Heartbeat finds an `in_progress` ticket with no live session |

Two-way coupling exists at these specific seams; otherwise the state machines don't share state.

## Operational consequences

- **`/acp status`** lists active *sessions* — answers "what's running right now?"
- **`nexus_status`** ([Nexus MCP](../components/nexus-mcp.md)) lists *tickets* across all companies — answers "what's the queue look like?"
- **`find_stuck_work`** queries *tickets* in flight beyond a threshold — uses ticket state, doesn't care about session identity
- The metrics DB joins both: "what's the median session duration *for tickets that ended in done*?" requires correlating session records to ticket transitions

A ticket can have a long history of attempts; a session is a snapshot of one of them.

## See also

- [Tickets](tickets.md) — the ticket state machine in detail (DFA 1)
- [Heartbeat](heartbeat.md) — the dispatch loop that creates the coupling
- [ACP plugin](../components/plugins/acp.md) — the runtime that owns sessions (DFA 4)
- [Postmortems](postmortems.md) — reference *sessions*, file follow-ups against *tickets*
- [Execution Layer](../architecture/execution-layer.md) — the architectural framing for both
