# Heartbeat

The **heartbeat** is the metronome of the platform. It runs on a fixed cadence (every 5 minutes by default), scans every company's backlog, and dispatches the highest-priority ready ticket for each.

## The loop

<img src="../../assets/diagrams/heartbeat-loop.svg" alt="Heartbeat dispatch loop — systemd timer triggers Global cap check; under-cap routes through List companies → Per-company filter → Pick top ticket → Spawn (brass), with loop-backs to List for skip / none / next-company. The over-cap branch goes straight to End cycle. List-exhausted also exits to End.">


The loop is *stateless across ticks* — the next tick rebuilds everything from Paperclip + the metrics DB. The only persistent state between ticks is what's in those stores.

## The loop, in pseudocode

```python
global_live = count_live_workers()
if global_live >= NEXUS_MAX_LIVE_WORKERS:   # default 1
    return

for company in companies:
    # per-company cap from session_allocations (default 2)
    if company.is_paused or company.in_flight >= company.session_cap:
        continue

    candidates = filter_backlog(company,
        eligible_priorities=peak_hour_filter(),
        unblocked_only=True,
    )

    if not candidates:
        continue

    ticket = highest_priority(candidates)
    spawn_agent(ticket)
    ticket.status = "in_progress"
```

Each iteration is **stateless** — there's no in-memory queue carried across runs. The current state of Paperclip + the postgres metrics DB is the only source of truth.

## What drives dispatch decisions

| Input | How it's used |
|---|---|
| **Backlog** | Set of tickets in `backlog` status, per company. |
| **Priority** | Tickets ordered by priority within a company. Higher priority first. |
| **Labels** | Some labels gate dispatch (e.g., `blocked-by-X`). |
| **Session cap** | Two layers. The **per-company** cap is read from the `session_allocations` table (default 2 if no row). The **global** cap is the env var `NEXUS_MAX_LIVE_WORKERS` (default `1` under the current Phase-1 ramp). Dispatch is gated by the *minimum* of the two. |
| **Peak hour filter** | During peak hours (configurable, e.g., 14:00-20:00 weekdays Europe/Amsterdam), only `high` and above run, with a reduced cap. |
| **Memory infra health** | If MemPalace API is down, all spawns skip this cycle. Recorded in the heartbeat log. |
| **Idle escalation flag** | If a company is idle for N cycles, optionally escalate (notify operator). Gated by `NEXUS_IDLE_ESCALATION_DISABLED`. |

## Why this pattern

A heartbeat is one of three common scheduling shapes for dispatch:

| Pattern | Tradeoff |
|---|---|
| **Webhook-triggered** | Lowest latency, but a missed event = lost dispatch. Requires event-store discipline. |
| **Event loop** (long-poll) | Low latency, but a daemon failure stalls everything. |
| **Heartbeat** (cron) | Higher latency (up to one cycle), but **self-healing** — a missed cycle is recovered next cycle, no event store needed. |

Nexus uses heartbeat as the **primary** dispatch path because it's the most failure-tolerant. The ACP plugin supplements this with webhook hooks for **lower-latency** dispatches on hot events (issue created, status change) — but heartbeat is always the safety net.

## Where heartbeat runs

Two options:

1. **systemd timer** (recommended) — runs every 5 min, persistent, auto-restarts. Defined in `~/.config/systemd/user/nexus-heartbeat.timer`.
2. **Manual loop** — `uv run python scripts/company_heartbeat.py` for ad-hoc invocation or development.

The timer respects `Persistent=true`, so if the host is asleep across a scheduled tick, it catches up on resume.

## Lock guard

The heartbeat is **idempotent at the cycle level** — running it twice in the same minute is safe. But Nexus also guards against forkbomb-shaped failures (where each heartbeat spawns a new worker without waiting for the previous one) at the **spawn target level**. Background spawns (like the memory promoter) check a PID-lockfile before launching to skip duplicates.

## Observability

Every heartbeat cycle logs:

- Companies considered + skipped (with reason)
- Tickets dispatched per company
- Idle-cycle counts (for escalation tracking)
- Memory infra health (probes MemPalace + Context-1)

Logs go to journalctl (when running under systemd) or stdout (manual).

## See also

- [Tickets](tickets.md) — what gets dispatched
- [Companies](companies.md) — what holds backlogs
- [Sessions vs. Tickets](sessions-vs-tickets.md) — why a spawn isn't the same as a ticket
- [Nexus Core](../components/nexus-core.md) — implementation
