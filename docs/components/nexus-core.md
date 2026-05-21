# Nexus Core

<p class="lede">Nexus Core is the orchestration engine — it picks tickets off backlogs, spawns agents, supervises sessions, runs the merge agent, and writes postmortems when things break. It's the concrete implementation of Nexus's <a href="../../architecture/execution-layer/">execution layer</a>.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-19</span>
  <span>Owner: Platform</span>
</div>

## What it is

A Python 3.11+ package providing both a CLI (`nexus`) and a set of library modules that other components import. The heartbeat runs as a `nexus-heartbeat.timer` systemd unit on a 5-minute cadence; other functionality (postmortems, merge agent, metrics) is invoked from within the heartbeat or directly by agents.

| Property | Value |
|---|---|
| **Path** | `~/Projects/nexus/nexus-core/` |
| **Language** | Python 3.11+ (uv-managed) |
| **Heartbeat** | `nexus-heartbeat.timer` (5-min cadence, systemd user unit) |
| **Entry point** | `scripts/company_heartbeat.py` |
| **Metrics DB** | PostgreSQL on the system instance (port `5432`), database `nexus_metrics`; configured via env var `NEXUS_METRICS_DB`. Separate from Paperclip's embedded Postgres on `54329`. |

## Modules

| Module | Job |
|---|---|
| `nexus/companies/` | Company provisioning from template repo + Paperclip + GitHub Project; session allocator (20 total, 2 reserved, 18 distributed). |
| `nexus/agents/` | Agent catalog loader (YAML → config), skill matching, agent provisioning. |
| `nexus/execution/` | Dispatch logic, session-complete handler, postmortem pipeline, merge agent. |
| `nexus/scheduler/` | Cron-style schedule resolution and `nexus-heartbeat` timer integration. |
| `nexus/routines/` | YAML-routine execution surface used by `routine-catalog` to call back into the substrate. |
| `nexus/meetings/` | Priority queue (P1 > P2 > P3 > P4), lifecycle state machine, token-efficient context passing for the facilitator. |
| `nexus/metrics/` | Performance records into PostgreSQL; KPI queries; pattern detection across agent types and versions. |
| `nexus/autoresearch/` | Background research pipeline — discovers candidate work, runs evaluators + mutators, feeds back into the catalog. |
| `nexus/deploy/` | Systemd-unit + venv install helpers used by `nexus-start` and per-host bootstrap. |

A `nexus/_deprecated/` directory also exists; it holds frozen-but-not-deleted modules retained for historical reference and is not on any import path.

## The dispatch loop

The heartbeat is the metronome:

```python
# scripts/company_heartbeat.py - runs every 5 minutes via systemd timer.
def main_loop():
    # 1. Honor the global concurrency cap.
    if count_live_workers() >= int(os.getenv("NEXUS_MAX_LIVE_WORKERS", "1")):
        return                                       # no headroom this cycle

    # 2. Per-company scan in priority order.
    for company in list_companies(sorted_by="priority"):
        if company.is_paused or company.in_flight >= company.session_cap:
            continue                                 # respect per-company cap

        # 3. Eligible-ticket filter (priority, labels, peak-hour gate).
        candidate = pick_next_ticket(company)
        if candidate is None:
            continue

        # 4. Atomic claim via Paperclip API.
        spawn_agent_for_ticket(candidate)            # transitions todo → in_progress
```

Pseudocode above; the real `company_heartbeat.py` adds telemetry, circuit-breaker checks, peak-hour filters, and idle-cycle tracking.

## Session lifecycle

When the heartbeat picks a ticket, the [ACP plugin](plugins/acp.md) takes over and runs the agent. Nexus Core watches the session via `on_session_complete`:

| Outcome | Nexus Core does |
|---|---|
| Agent exited 0 + posted verdict comment | Transitions ticket via Paperclip API (`in_progress → in_review` or onward) |
| Agent exited 0 + silent | Treats as silent termination; retries via the review-escalation DFA |
| Agent exited non-zero | Records failure, increments circuit-breaker; may trigger postmortem |
| Timeout (idle 30m or max-age 8h) | Kills session, marks ticket for retry |

## Postmortem pipeline

Failures that match "interesting" patterns trigger the postmortem pipeline (DFA 20 in `docs/state-machines.md`):

```mermaid
flowchart LR
    FAIL[Failure detected] --> BUILD[Build record]
    BUILD --> SCAN[Scan eval gaps]
    SCAN --> ADR[Write ADR]
    ADR --> FILE[File follow-up ticket]

    classDef step fill:#0D0F11,stroke:#6F7177,color:#E6E3DC,stroke-width:1px
    class FAIL,BUILD,SCAN,ADR,FILE step
```

Postmortems live in the relevant repo at `docs/postmortems/<id>.md`. They're never deleted — old ones are retained as history.

See [Triage a Postmortem](../guides/triage-a-postmortem.md) for how to action one.

## The merge agent

When a review verdict is `approve`, Nexus Core triggers the merge agent (DFA 3):

1. **Preconditions** — branch_ref exists, approve verdict on record, no active merge lock on repo
2. **Checkout main** — switch to `main`, pull latest
3. **Fast-forward or rebase** — try fast-forward first; rebase if not possible
4. **Test gate** — run the project's test suite; fail fast on red
5. **Push** — once, with retry on transient failure
6. **Prune** — delete local + remote branch

Failure at any step ends in `refused` / `conflict` / `test_failed` / `error` (per DFA 3). The merge agent never overrides a failure — that's a human decision.

## Session allocator

Nexus runs with a finite session budget — 20 concurrent sessions across the org by default:

| Pool | Count | Notes |
|---|---|---|
| Parent company | 2 | Reserved for board meetings, governance work |
| Companies | 18 | Dynamically rebalanced by the COO agent |
| **Total** | **20** | P1-priority tickets can preempt non-critical slots |

The Phase-1 ramp sets `NEXUS_MAX_LIVE_WORKERS=1` as a global cap, well below the 20 total. This is intentional while we iterate on stability.

## Configuration

Environment variables (set by the systemd unit or shell):

```bash
# Metrics DB connection (system Postgres on :5432, database nexus_metrics)
NEXUS_METRICS_DB=postgresql:///nexus_metrics

# Paperclip API base (used by company_heartbeat + provisioners)
NEXUS_PAPERCLIP_API_BASE=http://127.0.0.1:3100/api

# Where the agent catalog YAMLs live
NEXUS_AGENT_CATALOG_PATH=/home/ian/Projects/nexus/agent-catalog/agents
NEXUS_AGENT_CATALOG_TEAMS_PATH=/home/ian/Projects/nexus/agent-catalog/teams

# Where the shared-skills directory lives
NEXUS_SHARED_SKILLS_DIR=/home/ian/Projects/nexus/shared-skills

# Local clone of the company-template repo (used by provisioning)
NEXUS_COMPANY_TEMPLATE_DIR=/home/ian/Projects/nexus/company-template

# Global concurrency cap (Phase-1 production: 1)
NEXUS_MAX_LIVE_WORKERS=1

# Require memory infra (MemPalace) to be reachable before dispatching
NEXUS_REQUIRE_MEMORY_INFRA=true

# Disable the "idle company" escalation notification
NEXUS_IDLE_ESCALATION_DISABLED=true

# GitHub token for repo + project operations (consumed by provision_company.py)
GITHUB_TOKEN=ghp_...
```

## Running it

### systemd timer (recommended)

```bash
# Start the timer (heartbeat fires every 5 minutes)
systemctl --user start nexus-heartbeat.timer

# Status
systemctl --user status nexus-heartbeat.timer

# Logs from the most recent fires
journalctl --user -u nexus-heartbeat.service -n 100
```

### Manual run

```bash
cd ~/Projects/nexus/nexus-core
uv run python scripts/company_heartbeat.py             # one cycle
uv run python scripts/company_heartbeat.py --wake      # force immediate scan
```

### Library use

```python
from nexus.companies import provisioner, session_allocator
from nexus.agents import catalog
from nexus.metrics import recorder, analyzer

# Used by other tools/scripts that want to integrate
```

## Setup

Requires Python 3.11+ and [uv](https://docs.astral.sh/uv/):

```bash
cd ~/Projects/nexus/nexus-core
uv venv
uv pip install -e ".[dev]"
uv run pytest                                          # verify install
```

## See also

- [Execution Layer](../architecture/execution-layer.md) — the architecture page this component implements
- [Heartbeat](../concepts/heartbeat.md) — concept-level description of the dispatch loop
- [Paperclip](paperclip.md) — what Nexus Core reads tickets from
- [ACP plugin](plugins/acp.md) — what actually spawns the agent processes
- [Nexus Memory](nexus-memory.md) — where session transcripts land after completion
- [Debug a Ticket](../guides/debug-a-ticket.md) — operational walkthrough
