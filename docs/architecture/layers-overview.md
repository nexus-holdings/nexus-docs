# Layers Overview

<p class="lede">Nexus is four layers, in a deliberate order, with strict boundaries. This page is the one-screen summary — what each layer owns, how they compose, where to dive deeper.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-19</span>
  <span>Owner: Platform</span>
</div>

## The four layers

<img src="../../assets/diagrams/layers-overview.svg" alt="Four-layer architecture overview — Governance at the top, Execution and Memory in the middle row, Platform (brass) as the foundation at the bottom. Solid arrows are writes/claims, dotted arrows are reads.">

The **platform layer** (brass-edged) is the foundation. The other three depend on it for identity and state; it depends on none of them. This is what makes Nexus *legible to agents* — there's one place to ask "what is the state of X?" and get a deterministic answer.

!!! note "On the four-layer framing"
    The canonical Nexus spec (`nexus-spec-1-0-final.md`) describes the system as an *"Autonomous AI Company Platform with C-suite governance, child companies, and shared catalogs"* — that's the official vocabulary. The four-layer model (Platform / Execution / Memory / Governance) used throughout these docs is a **pedagogical abstraction** over the implementation, not a separate naming scheme. Every primitive it names exists in the system; we just group them this way to make the dependency rules easier to teach.

## What each layer owns

| Layer | Owns | Doesn't own | Component |
|---|---|---|---|
| **[Platform](platform-layer.md)** | Entity state — companies, tickets, agents, plugins, contracts | The work that produces them | [Paperclip](../components/paperclip.md) |
| **[Execution](execution-layer.md)** | Turning intent into action — dispatch, sessions, merge, postmortems | Whether work is "good" | [Nexus Core](../components/nexus-core.md) + [ACP plugin](../components/plugins/acp.md) |
| **[Memory](memory-layer.md)** | Persisting and retrieving content that survives a session | Reasoning about that content | [Nexus Memory](../components/nexus-memory.md) |
| **[Governance](governance.md)** | Evals, postmortems, ADRs, contracts | The artifacts' storage | Distributed — [Eval Registry](../components/eval-registry.md), Contracts plugin, Nexus Core postmortem pipeline |

The "doesn't own" column is as load-bearing as the "owns" column. Every layer's boundary is enforced by what it *refuses* to do.

## How a ticket flows through all four

A worked example, in order:

1. **Platform** — a ticket is filed. Paperclip stores it with status `backlog`. Owner: a specific company.
2. **Execution** — the next heartbeat picks it up, transitions `backlog → todo → in_progress`, spawns an implementer agent.
3. **Memory** — when the agent finishes, the session transcript is ingested into MemPalace (wing `claude`, room `sessions`).
4. **Governance** — the reviewer agent runs the relevant evals, posts a verdict. If something failed in an interesting way, a postmortem is filed.
5. **Platform** — the ticket transitions to `done` (or back to `todo` on reject). Status visible to anyone with access.

Each layer is responsible for one job in this flow. None of them does someone else's.

## The dependency graph (one rule)

> **Lower layers don't read higher layers.**

Concretely:

- Platform layer doesn't read execution, memory, or governance
- Execution reads platform (to find tickets) and writes to memory (transcripts); doesn't read memory
- Memory reads from platform (to scope drawers by company/ticket); doesn't read execution
- Governance reads from all three (to score work, build postmortems, write ADRs); writes only judgments

If you find a code path where a lower layer reads from a higher layer, that's a smell. Either the abstraction is wrong or there's a circular dependency hiding.

## Where to dive next

- Read [The Flywheel](flywheel.md) first if you want the *why* — the load-bearing thesis the four layers exist to support
- Read [Substrate vs Flywheels](substrate-vs-flywheels.md) for the positioning — why Nexus is shaped as a substrate, not a framework
- Then drill into individual layers in any order:
  - [Platform Layer](platform-layer.md)
  - [Execution Layer](execution-layer.md)
  - [Memory Layer](memory-layer.md)
  - [Governance](governance.md)

If you're operationally minded and want to see what each layer *looks like* as code, the [Components](../components/paperclip.md) section has one page per implementation.

## See also

- [The Flywheel](flywheel.md) — the substrate-plus-flywheels thesis
- [Substrate vs Flywheels](substrate-vs-flywheels.md) — why this shape vs alternatives
- [Components index](../components/paperclip.md) — what implements each layer
