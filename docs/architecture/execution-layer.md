# Execution Layer

<p class="lede">The execution layer turns intent into action. It takes <code>todo</code> tickets from the platform layer, spawns agents against them, watches sessions run, posts verdicts, merges code, and writes postmortems when things break. It's where the platform's tracked entities actually <em>do work</em>.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-19</span>
  <span>Owner: Platform</span>
</div>

## What it does

Five responsibilities, in order of the work they handle:

1. **Dispatch** — scan company backlogs, pick the highest-priority eligible ticket, spawn an implementer agent
2. **Session management** — start, observe, and clean up agent processes; route stdin/stdout; enforce timeouts
3. **Review orchestration** — when implementation finishes, spawn a reviewer agent against the same ticket
4. **Merge** — on an approved review, run the merge agent to land the branch on `main`
5. **Postmortem** — when something fails (zombie sessions, repeated rejects, merge conflicts), build a structured failure record

Together these wrap a `todo` ticket and either return `done`, kick it back to `todo` for rework, or escalate it.

## The dispatch loop

The execution layer's heartbeat fires every five minutes. Pseudocode:

```python
# 1. Honor the global concurrency cap before doing any work.
global_live = count_live_workers()                          # how many ACP sessions are running
if global_live >= NEXUS_MAX_LIVE_WORKERS:                   # env-level kill switch (default 1 in Phase 1)
    return                                                  # no headroom — skip this cycle

# 2. Walk each company in priority order.
for company in companies:
    if company.is_paused or company.in_flight >= company.session_cap:
        continue                                            # respect per-company cap

    # 3. Filter the backlog by what's eligible to dispatch right now.
    candidates = filter_backlog(
        company,
        eligible_priorities=peak_hour_filter(),             # peak hours: high+ only
        unblocked_only=True,                                # skip `blocked` tickets
    )
    if not candidates:
        continue

    # 4. Claim the top ticket atomically via the platform API.
    ticket = highest_priority(candidates)
    spawn_agent(ticket)                                     # ACP plugin handles the subprocess
    ticket.status = "in_progress"                           # platform-layer state transition
```

Every step is idempotent and stateless across cycles. If the host sleeps through a tick, the next tick picks up where the missed one would have. This is the "self-healing" property the heartbeat trades latency for.

## The four state machines

The execution layer is the union of four formal DFAs. Each is documented in detail in `docs/state-machines.md`; this is the map.

| DFA | What it governs | Where it lives |
|---|---|---|
| **1 — Ticket lifecycle** | The end-to-end state of a unit of work | Source of truth on platform layer; execution drives transitions |
| **4 — Session lifecycle** | A single ACP agent process from spawn to close | ACP plugin (`paperclip-plugin-acp`) |
| **5 — Heartbeat orchestration** | The dispatch scanner's own loop | `nexus-core/scripts/company_heartbeat.py` |
| **6 — Scheduler dispatch cycle** | Per-company picking + claim atomicity | `nexus-core/nexus/execution/` |

Tickets are the dominant entity; sessions are the work units; heartbeat is the metronome; dispatch is the per-cycle decision logic.

## Session lifecycle

When the heartbeat picks a ticket, the [ACP plugin](../components/plugins/acp.md) takes over:

<img src="../../assets/diagrams/execution-session-states.svg" alt="Session lifecycle state machine (DFA 4) — spawning → idle → processing along the top with a turn_complete loop back from processing to idle. Error states (spawn_fail → error, idle → closing on timeout) drop to a bottom row that converges on closed (brass, accept state). exit_0 / exit_nonzero routes from processing direct to closed.">


Three boundaries the execution layer enforces:

- **Idle timeout** — 30 minutes of silence ⇒ kill the session
- **Max age** — 8 hours wall-clock ⇒ kill the session
- **Cancel** — operator or escalation can SIGINT a running session at any time

On normal exit, the session manager:

1. Captures the final transcript (lands in [Nexus Memory](../components/nexus-memory.md))
2. Reads the agent's completion comment for branch + SHA
3. Transitions the ticket to `in_review` (or back to `in_progress` on failure-with-retry)
4. Releases the session-cap slot

## Review orchestration

When a ticket enters `in_review`, the execution layer spawns a reviewer agent. The review state machine (DFA 2) has three escalation tiers:

| Attempt | Model | Notes |
|---|---|---|
| 1 | Sonnet | Default — fast and cheap |
| 2-3 | Opus | Escalation after silent termination |
| > 3 | Chairman | Manual review path |

A reviewer must exit with a valid verdict comment (approve / reject + reasoning). A "silent termination" — exit code 0 but no verdict — counts as a failure for retry purposes. There's a known gap (NEXA-65) where a missed webhook can leave a ticket stuck in `in_review` with no spawn; the long-term fix is a staleness alert.

## The merge agent

On an approved review, the merge agent (DFA 3) handles the actual git operations:

1. **Preconditions** — ticket has a branch_ref artifact; an approved review verdict is on record; no active merge lock on the repo
2. **Checkout main** — switch local repo to `main`, pull latest
3. **Merge or rebase** — fast-forward if possible, otherwise rebase the branch
4. **Test gate** — run the project's test suite; fail fast on red
5. **Push** — once with a retry on transient failure
6. **Prune** — delete local + remote branch on success

Any failure (conflict, test red, push fail after retry) emits a structured event that the postmortem pipeline picks up. The merge agent never overrides a failure — that's a human decision.

## Postmortem on failure

When the execution layer detects an "interesting failure" — repeated rejects, merge conflict, timeout, zombie session, circuit breaker trip — it kicks off the postmortem pipeline (DFA 20):

1. Build a failure record with ticket, session, transcript, and diffs attached
2. Scan recent evals for a matching gap (was this failure mode covered by any rubric?)
3. Write an ADR or update an existing one
4. File a follow-up ticket (often labelled `eval-gap` or `infra-fix`)

The output of the postmortem is what feeds back into [Governance](governance.md) — the eval registry grows, the ADR set deepens, the next dispatch in the same shape has a better chance.

## Self-healing as a design principle

The execution layer chose heartbeat over webhook-primary on purpose. Three options exist:

| Pattern | Latency | Failure mode |
|---|---|---|
| **Webhook-triggered** | Lowest (sub-second) | A missed event = a lost dispatch; needs an event store + replay |
| **Event loop (long-poll)** | Low (seconds) | Daemon crash stalls everything until restart |
| **Heartbeat (cron)** | Higher (one cycle) | Missed cycle is recovered next cycle — **self-healing** |

Nexus uses heartbeat as the **primary** dispatch path because it's the most failure-tolerant. The [ACP plugin](../components/plugins/acp.md) supplements with webhook hooks for lower-latency response on hot events (issue created, status change) — but heartbeat is always the safety net.

## What it does not do

The execution layer **does not**:

- **Track entity state** — that's the [platform layer](platform-layer.md). The execution layer reads + transitions state via API; it doesn't own the records.
- **Decide whether work is good** — that's [governance](governance.md). The execution layer only knows whether a session exited 0 and whether the merge agent succeeded.
- **Persist agent output** — that's [memory](memory-layer.md). The execution layer hands off the transcript on session exit; what happens to it afterwards is memory's concern.

This separation is what keeps the dispatch loop short enough to be auditable.

## Implementation

The execution layer is implemented by two components:

- **[Nexus Core](../components/nexus-core.md)** — `company_heartbeat.py`, the dispatch decision logic, postmortem pipeline, merge agent
- **[ACP plugin](../components/plugins/acp.md)** — the session-management surface (spawn, send, cancel, close)

Configuration:

```bash
# Maximum concurrent ACP sessions across all companies. Phase-1 default = 1.
NEXUS_MAX_LIVE_WORKERS=1

# Require memory infra (MemPalace API) to be reachable before dispatching.
NEXUS_REQUIRE_MEMORY_INFRA=1

# Disable the "idle company" escalation notification.
NEXUS_IDLE_ESCALATION_DISABLED=1
```

Heartbeat is run as a systemd timer (`nexus-heartbeat.timer`, 5-min cadence) in production, or by `uv run python scripts/company_heartbeat.py` for ad-hoc invocation.

## See also

- [Heartbeat](../concepts/heartbeat.md) — the dispatch loop in detail
- [Tickets](../concepts/tickets.md) — the entity being dispatched
- [Nexus Core](../components/nexus-core.md) — the implementation
- [ACP plugin](../components/plugins/acp.md) — session management
- [Platform Layer](platform-layer.md) — what the execution layer reads from
- [Memory Layer](memory-layer.md) — where transcripts land
- [Governance](governance.md) — what reacts to failures
