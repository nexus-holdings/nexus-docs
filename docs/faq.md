# FAQ

<p class="lede">Answers to the questions people ask in their first week with Nexus. This page is a router — each answer is short and points at the doc where the topic is explained properly.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-20</span>
  <span>Owner: Platform</span>
</div>

## Getting oriented

### What is Nexus, in one paragraph?

Nexus is the persistent substrate that agents run on — a platform/execution/memory/governance stack that tracks tickets, dispatches agents on a heartbeat, captures every session into long-term memory, and records load-bearing decisions as ADRs. It is not a single product; it is the layer that any number of self-improving agentic verticals (called *flywheels*) ride on. See [The Flywheel](architecture/flywheel.md) for the load-bearing thesis.

### Is Nexus a framework, a platform, or a service?

A platform. The thesis is set out in [Substrate vs. Flywheels](architecture/substrate-vs-flywheels.md): Nexus is the part that persists, gets stronger every time a flywheel turns, and is reused across verticals. Frameworks are libraries you call into; Nexus is a substrate your work happens *on top of*, with state machines, a control plane, and an opinionated memory architecture.

### How is Nexus different from LangChain / LlamaIndex / CrewAI?

Those are agent toolkits — libraries you import to build an application. Nexus is the substrate that runs *underneath* whatever you build: it provides the [companies](concepts/companies.md), [tickets](concepts/tickets.md), [heartbeat](concepts/heartbeat.md), and [memory](components/nexus-memory.md) so that agents have somewhere to live, queue work, and accumulate context across sessions. You can build a Nexus flywheel that uses any of those libraries inside it.

### Do I need Claude Code to use Nexus?

No. Claude Code is one of several coding agents the [ACP plugin](components/plugins/acp.md) can spawn — Codex, Gemini CLI, and OpenCode are also supported. Non-coding agents (reviewers, routine workers, plugin-defined tool agents) don't go through ACP at all and have no Claude Code dependency.

## Concepts

### What's the difference between a company and a wing?

A [company](concepts/companies.md) is an *organisational* unit — it owns a backlog of tickets and runs agents to drain it. A [wing](concepts/wings-and-rooms.md) is a *memory* unit — it partitions MemPalace so a company's drawers don't bleed into another's. Most companies have a matching wing, but the two concepts are independent: a wing can outlive its company; a company can read from wings it does not own.

### What's the difference between an agent and a session?

An [agent](concepts/agents.md) is a *configured worker* — a prompt + tool list + model, defined once and reused. A [session](concepts/sessions-vs-tickets.md) is one *run* of an agent against a specific ticket; it produces a transcript, a verdict, and a memory write, then closes. One ticket can outlive many sessions; one agent can have run thousands of sessions. They are separate state machines on purpose — see [Sessions vs Tickets](concepts/sessions-vs-tickets.md) for the full breakdown.

### When should I use a contract vs a craft-dispatch ticket?

Use [craft-dispatch](components/plugins/craft-dispatch.md) for routine cross-company work where the obligation is implicit — a domain company asking Nexus Engineering for a standard implementation, for instance. Use a [contract](concepts/contracts.md) when the agreement has *terms* worth tracking formally: scope, acceptance criteria, deadlines, or an external counterparty. The contract holds the obligation; the linked tickets do the work.

### Why does Nexus use a heartbeat instead of webhooks?

Because heartbeats are *failure-tolerant*: if the previous cycle crashed, the next one just picks up where it left off, with no missed events. Webhooks lose work when the receiver is down. The [Heartbeat](concepts/heartbeat.md) page has the full tradeoff table — Nexus uses heartbeat as the *primary* dispatch path, with the ACP plugin supplementing it with webhook hooks for lower-latency hot events. Heartbeat is always the safety net.

### What are domain vs craft companies?

[Two-class companies](concepts/two-class-companies.md) is the load-bearing organising principle. A *domain* company owns a product or problem space and dispatches well-formed tickets; a *craft* company is a stateless execution engine for one capability (e.g., Nexus Engineering for code). One engineering craft serves many product domains, the same way one engineering department serves many product teams in a traditional company. ADR-032 is the source decision.

## Operations

### Why is my ticket stuck in `in_progress`?

Usually one of three things: the session crashed without flipping state (zombie — the heartbeat will reset it on the next tick), the agent is waiting on an external dependency (blocked, but the ticket wasn't moved to `blocked`), or the global session cap is full and a new spawn is being skipped. Run through [Debug a Ticket](guides/debug-a-ticket.md) for the diagnostic walk-through.

### How do I dispatch work to a craft company?

Call the `craft_dispatch_ticket` tool from a domain agent. The [Craft Dispatch plugin](components/plugins/craft-dispatch.md) translates that into a well-formed ticket on the craft company's backlog, records `origin_kind` + `origin_id` for traceability, and arranges flow-back so the source ticket gets updated when the craft work finishes. If the work has formal terms, wrap it in a [contract](concepts/contracts.md) first.

### What happens when an agent crashes mid-session?

The session DFA records the failure and closes; the [ticket](concepts/tickets.md) it was working on is left in `in_progress`. On the next [heartbeat](concepts/heartbeat.md) cycle, the orphan-recovery job detects the inconsistency and resets the ticket back to `todo` so another session can pick it up. Repeated crashes trip the per-agent circuit breaker and pause that agent until a human re-enables it. The crash itself becomes a [postmortem](concepts/postmortems.md) if the failure was non-trivial.

### How do I find out which model an agent used for a specific ticket?

Sessions are pinned to their ticket and store the full agent config including model. The Paperclip API (`/v1/sessions?ticket_id=...`) returns the chain of sessions for a ticket, each with its agent id and model; the [Agent Catalog](components/agent-catalog.md) explains how agents are configured. The session transcript ingested into [memory](concepts/ingestion.md) also captures this — Lighthouse-style queries can surface it without going through the API.

## Build and extend

### How do I add a new agent type?

Walk through [Create an Agent](guides/create-an-agent.md). The summary: define the agent in the [Agent Catalog](components/agent-catalog.md) with its prompt, tool list, and model, declare which company owns it, and (if it needs a new capability) wire that capability through the relevant plugin. Tests + an eval entry are required before the agent is allowed to dispatch.

### How do I write a Paperclip plugin?

The [Create a Plugin](guides/create-a-plugin.md) guide is the entry point. Plugins are typed extensions to Paperclip — they contribute workers, tools, and config; the [ACP](components/plugins/acp.md), [Contracts](components/plugins/contracts.md), and [Craft Dispatch](components/plugins/craft-dispatch.md) plugins are good reference implementations to read alongside the guide.

### Can I run Nexus without a GPU?

Yes, with a caveat. The [Nexus Memory](components/nexus-memory.md) embedder (bge-m3) defaults to CUDA but falls back to CPU when you set `BGE_M3_DEVICE=cpu`. Promotion throughput drops substantially in that mode — it's fine for development and small personal corpora; not recommended for a production substrate driving multiple flywheels.

## See also

- [Cheat Sheet](getting-started/cheat-sheet.md) — one-line glossary of every term used in the docs
- [Layers Overview](architecture/layers-overview.md) — the four-layer map of the substrate
- [Decisions Index](concepts/decisions-index.md) — every load-bearing decision as a numbered ADR
- [Roadmap](roadmap.md) — what's shipped, what's in flight, what's planned
