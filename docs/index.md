---
hide:
  - navigation
  - toc
---

# Nexus Docs

<p class="lede">Nexus is a substrate for running <strong>companies of AI agents</strong>: a control plane that owns the work (companies, tickets, agents), a memory layer that survives every session, and a governance layer — contracts, goals, evals, human gates — that lets the system improve without drifting. These docs are for the people building on top.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-19</span>
  <span>Owner: Platform</span>
</div>

<div class="grid cards" markdown>

-   <span class="kicker">01 · Start</span>
    <span class="card-title">[Quickstart](getting-started/overview.md)</span>

    Install the stack, file a ticket, watch an agent carry it to done.

-   <span class="kicker">02 · Concepts</span>
    <span class="card-title">[The control plane](concepts/companies.md)</span>

    Companies own the work, tickets carry it, agents execute it. Start here — every other page assumes these three.

-   <span class="kicker">03 · Concepts</span>
    <span class="card-title">[Memory model](concepts/wings-and-rooms.md)</span>

    Wings, rooms, drawers: where knowledge lives, who can see it, and how it finds its way back into a session.

-   <span class="kicker">04 · Architecture</span>
    <span class="card-title">[The flywheel](architecture/flywheel.md)</span>

    Why outcomes compound back into the substrate instead of plateauing the way pipelines do.

-   <span class="kicker">05 · Reference</span>
    <span class="card-title">[API](reference/api-paperclip.md)</span>

    Every endpoint, parameter, and error — control plane and memory, with curl examples.

-   <span class="kicker">06 · Internals</span>
    <span class="card-title">[Execution stack](components/nexus-core.md)</span>

    How a ticket becomes a running agent session: dispatch, heartbeat, and the audit trail it leaves.

</div>

## The mental model

<img src="assets/diagrams/index-substrate-flywheels.svg" alt="Nexus mental model — substrate (brass-bordered) as a full-width foundation at the bottom, three flywheel columns rising above (Aurelius as the canonical example, plus two further verticals). Bidirectional arrows in each gap show services going up and outcomes flowing back down.">

The brass-bordered foundation is Nexus. Each column rising from it is a separate vertical product — a flywheel — running on the substrate: services flow up, outcomes flow back down, and every outcome that lands makes the substrate (and the next flywheel built on it) better.

The substrate's primitives are deliberately few: [companies](concepts/companies.md) own backlogs, [tickets](concepts/tickets.md) are the unit of work, [agents](concepts/agents.md) execute them, [contracts](concepts/contracts.md) make cross-company obligations explicit, [goals](concepts/goal-aware-coordination.md) keep both sides of a contract pointed the same way, and the [flywheel](architecture/flywheel.md) compounds the outcomes back in. New companies are born from approved specs through a human-gated [spawn pipeline](components/plugins/agora.md) — the org chart itself is a substrate artifact.
