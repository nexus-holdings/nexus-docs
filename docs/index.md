---
hide:
  - navigation
  - toc
---

# Nexus Docs

<p class="lede">A control plane, persistent memory, and execution stack for self-improving agentic systems. Documentation for the people building on top.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-19</span>
  <span>Owner: Platform</span>
</div>

<div class="grid cards" markdown>

-   <span class="kicker">01 · Start</span>
    <span class="card-title">[Quickstart](getting-started/overview.md)</span>

    From zero to a running ticket and back, end-to-end.

-   <span class="kicker">02 · Concepts</span>
    <span class="card-title">[The control plane](concepts/companies.md)</span>

    Companies, tickets, agents — the primitives every other page assumes.

-   <span class="kicker">03 · Concepts</span>
    <span class="card-title">[Memory model](concepts/wings-and-rooms.md)</span>

    Wings, rooms, drawers. How knowledge is stored and scoped.

-   <span class="kicker">04 · Architecture</span>
    <span class="card-title">[The flywheel](architecture/flywheel.md)</span>

    Why the substrate-plus-flywheels shape compounds where pipelines plateau.

-   <span class="kicker">05 · Reference</span>
    <span class="card-title">[API](reference/api-paperclip.md)</span>

    All endpoints, parameters, errors.

-   <span class="kicker">06 · Internals</span>
    <span class="card-title">[Execution stack](components/nexus-core.md)</span>

    The runtime, the dispatch loop, and the audit trail.

</div>

## The shape

<img src="assets/diagrams/index-substrate-flywheels.svg" alt="Nexus mental model — substrate (brass-bordered) as a full-width foundation at the bottom, three flywheel columns rising above (Aurelius as the canonical example, plus two further verticals). Bidirectional arrows in each gap show services going up and outcomes flowing back down.">

Nexus is the box on the left. Each flywheel on the right is a separate vertical product running on top of the substrate, exchanging outcomes back so the substrate (and the next flywheel that runs on it) gets better.

The substrate's primitives are simple: [companies](concepts/companies.md) own backlogs, [tickets](concepts/tickets.md) are the unit of work, [agents](concepts/agents.md) execute them, and the [flywheel](architecture/flywheel.md) is what compounds the outcomes back into the substrate.
