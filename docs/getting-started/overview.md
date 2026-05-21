# Overview

Nexus is a **substrate** — a platform that AI products run on top of. It provides the shared services that every long-running AI system ends up needing, so each product can focus on its domain instead of rebuilding identity, dispatch, memory, and governance from scratch.

!!! note "A note on examples"
    These docs use illustrative names — companies like `Aurelius`, `Lighthouse`, `Ledgerly`; agents like `chairman`, `coo`, `spec-writer`; wings like `aurelius` — to make commands and tables concrete. **Treat them as placeholders.** They don't exist on a fresh install; you'll create your own companies, agents, and wings as you go. The *syntax* shown around each placeholder is real — substitute your own slugs and the commands will work.

## Mental model

<img src="../../assets/diagrams/overview-mental-model.svg" alt="Nexus mental model — substrate as a foundation at the bottom (brass), holding the four layers (Platform, Execution, Memory, Governance) as inner boxes, with flywheel columns (placeholder verticals) rising out of it. Bidirectional arrows between each flywheel and the substrate show services going down and outcomes coming back up.">

Four primary layers sit beneath every product:

| Layer | Job | Component |
|---|---|---|
| **Control plane** | Tracks tickets, companies, agents, plugins | [Paperclip](../components/paperclip.md) |
| **Execution** | Picks tickets off backlogs, spawns agents, handles postmortems | [Nexus Core](../components/nexus-core.md) |
| **Memory** | Long-term knowledge store with semantic search | [Nexus Memory](../components/nexus-memory.md) |
| **UI** | Dashboard for monitoring + governance | [Cockpit](../components/cockpit.md) *(original UI archived; replacement being built as a plugin inside the governance company)* |

## What you do, vs. what Nexus does

| You | Nexus |
|---|---|
| File a ticket with title + acceptance criteria | Picks it up on the next heartbeat, spawns the right agent |
| Define an agent (prompt + tools) once | Re-uses it across every ticket that fits |
| Write a routine (cron-style spec) | Schedules + runs it, retries on failure |
| Capture an insight in a session | Auto-ingests it into long-term memory, makes it searchable |

## Two organizing primitives

Everything in Nexus is a **company** doing **tickets**. Both terms are slightly bent from their everyday meaning:

- **Company** — a unit of work scope. A product, a domain, a service, or a craft (like "engineering" or "research"). See [Companies](../concepts/companies.md).
- **Ticket** — a single piece of work with a clear acceptance criterion. Lifecycle: `backlog → todo → in_progress → in_review → done`. See [Tickets](../concepts/tickets.md).

These two abstractions cover the entire surface area: every agent spawn, every routine run, every memory write is keyed to a (company, ticket) pair.

## Where to go next

- [Installation](installation.md) — get the stack running locally
- [First Ticket](first-ticket.md) — file one end-to-end
- [Cheat Sheet](cheat-sheet.md) — glossary of terms
- [The Flywheel](../architecture/flywheel.md) — why the substrate-plus-flywheels shape compounds
