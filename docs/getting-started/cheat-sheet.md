# Concepts Cheat Sheet

Glossary of Nexus terminology, intentionally compressed. Each entry links to the page where it's explained properly.

## Workflow primitives

| Term | One-line definition |
|---|---|
| **[Company](../concepts/companies.md)** | A unit of work scope. Has a backlog and an identity. Two classes: *domain* (owns context) and *craft* (executes standardized work). |
| **[Ticket](../concepts/tickets.md)** | A single piece of work. States: `backlog → todo → in_progress → in_review → done`. `done` means merged to main. |
| **[Agent](../concepts/agents.md)** | A configured worker (prompt + tools + model). Lives in the [Agent Catalog](../components/agent-catalog.md). |
| **Team** | A named group of agents that work a ticket together (e.g., implementer + reviewer + merge-agent). |
| **[Routine](../components/routine-catalog.md)** | A scheduled recurring task. Cron-like spec. Lives in the [Routine Catalog](../components/routine-catalog.md). |
| **[Session](../concepts/sessions-vs-tickets.md)** | One agent invocation. Produces a transcript that gets ingested into memory. |
| **Plugin** | A typed extension to Paperclip with workers + tools (e.g., `paperclip-plugin-acp`). |
| **[Contract](../concepts/contracts.md)** | First-class inter-company agreement — scope, acceptance criteria, and a status lifecycle. |
| **[Skill](../components/skills-catalog.md)** | Reusable workflow definition expressed as a SKILL.md manifest that any agent can invoke. |
| **[Eval](../concepts/evals.md)** | Versioned scoring rubric; turns "did this look good?" into a queryable, comparable number. |

## Architecture

| Term | One-line definition |
|---|---|
| **[Substrate](../architecture/flywheel.md)** | Nexus itself — the persistent platform/execution/memory/governance layer that flywheels run on. |
| **[Flywheel](../architecture/flywheel.md)** | A self-improving agentic vertical riding on the substrate (Aurelius, Lighthouse, Ledgerly, …). |
| **[Two-class companies](../concepts/two-class-companies.md)** | Every company is either *domain* (owns context) or *craft* (stateless execution); ADR-032. |
| **[Layer](../architecture/layers-overview.md)** | Pedagogical grouping of the substrate: Platform / Execution / Memory / Governance. |
| **DFA** | Deterministic finite automaton; canonical reference for substrate state machines — see [Decisions Index](../concepts/decisions-index.md) and `docs/state-machines.md`. |

## Execution

| Term | One-line definition |
|---|---|
| **[Heartbeat](../concepts/heartbeat.md)** | Cron job (5-min cadence) that picks ready tickets off backlogs and spawns agents. |
| **Dispatch** | The act of moving a ticket from `todo` to `in_progress` by spawning an agent. |
| **Session cap** | Two layers. Per-company cap = DB-driven via `session_allocations` (default 2). Global cap = env var `NEXUS_MAX_LIVE_WORKERS` (default 1 in current Phase-1 ramp). Effective cap = min of the two. |
| **[Postmortem](../concepts/postmortems.md)** | Auto-generated forensic record when a ticket fails or completes oddly. Lands in `docs/postmortems/`. |
| **Circuit breaker** | Per-agent retry limit. After N consecutive failures, the agent is paused until manually re-enabled. |
| **[ACP](../components/plugins/acp.md)** | Agent Client Protocol — Paperclip runtime that spawns coding-agent subprocesses (Claude Code, Codex, …). |
| **[Craft Dispatch](../components/plugins/craft-dispatch.md)** | Cross-company ticket bridge; lets a domain company hand work to a craft company in one tool call. |
| **[Flow-back](../components/plugins/craft-dispatch.md)** | Status update from the craft ticket back to the source ticket on a final transition. |
| **[Verdict](../concepts/sessions-vs-tickets.md)** | Reviewer's `approve` / `reject` / `escalate` decision on a session's output. |

## Memory

| Term | One-line definition |
|---|---|
| **MemPalace** | Long-term knowledge store. Writes go through `/v1/remember`. |
| **Context-1** | Searchable, embedding-backed mirror of MemPalace. Reads go through `/v1/retrieve`. |
| **Promoter** | Background job that mirrors MemPalace → Context-1. Runs incrementally. |
| **[Wing](../concepts/wings-and-rooms.md)** | Top-level memory partition (e.g., `claude`, `nexus`, `project-X`). |
| **[Room](../concepts/wings-and-rooms.md)** | Sub-partition within a wing (e.g., `sessions`, `decisions`). |
| **Drawer** | A single memory item. Content-hashed for dedup. |
| **[Ingestion](../concepts/ingestion.md)** | Pipeline that captures content (sessions, files, code, diary) into MemPalace via a single write path. |
| **[bge-m3](../components/nexus-memory.md)** | GPU embedder used by the promoter; falls back to CPU via `BGE_M3_DEVICE=cpu`. |
| **[AAAK](../concepts/aaak.md)** | *Deprecated.* Compressed dialect originally proposed for structured facts. Defined but not in active use — kept for protocol reference. |

## Governance

| Term | One-line definition |
|---|---|
| **[ADR](../concepts/decisions-index.md)** | Architectural Decision Record. Numbered, never deleted; the canonical record of a load-bearing decision. |
| **[Postmortem](../concepts/postmortems.md)** | Forensic record of a ticket failure or odd completion; feeds the gap-scan. |
| **[Gap-scan](../concepts/postmortems.md)** | Keyword-match a postmortem's root cause against eval dimensions; surfaces missing evals. |
| **[Self-improvement](../concepts/two-class-companies.md)** | The domain company that owns evals and cross-company metrics analysis. |

## Infrastructure

| Term | One-line definition |
|---|---|
| **Paperclip** | Control plane API + UI. Port 3100. |
| **Cockpit** | *Pending rewrite.* Original Next.js operator dashboard at port 3000 is archived; replacement is being built as a plugin inside the governance company. |
| **ChromaDB** | Vector database backing Context-1. Port 8101. |
| **MemPalace API** | REST surface in front of MemPalace + Context-1. Port 8102. |
| **MCP server** | Model Context Protocol bridge into Nexus from external AI clients. |
| **Heartbeat service** | Systemd timer that fires the dispatch loop. |
| **Promoter service** | Systemd timer that runs the MP→C1 promoter. |

## Status & gating

| Term | One-line definition |
|---|---|
| **Skill flag** | Boolean capability check before an agent attempts a task (e.g., `can_merge`, `can_review`). |
| **Eval gap** | Coverage gap detected by the eval registry — a class of work without an eval spec. |
| **Backlog** | The queue of `backlog`-status tickets for a company. Heartbeat prioritizes here. |
| **In-flight** | Tickets in `in_progress` or `in_review`. Count toward session cap. |
| **Orphan** | A ticket whose session died without transitioning state. Recovery job re-queues it. |
