# Roadmap

<p class="lede">A curated view of what has recently landed in Nexus, what is actively in flight, and what is on the planning surface. The roadmap is intentionally short — the full ADR stream is the ground truth.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-20</span>
  <span>Owner: Platform</span>
</div>

## Recently shipped

Roughly the last few weeks. Each item links to the ADR or component page that records the change.

- **ADR-032 accepted — [Two-class companies](concepts/two-class-companies.md).** Domain vs. craft companies are now the load-bearing organising principle of the substrate.
- **ADR-037 accepted — Cross-company dispatch.** Formalised the protocol that the [Craft Dispatch plugin](components/plugins/craft-dispatch.md) implements.
- **ADR-043 accepted — [Contracts](concepts/contracts.md) as first-class primitive.** Inter-company agreements moved from implicit (two columns on a dispatched ticket) to explicit (their own state machine).
- **`paperclip-plugin-contracts` shipped.** Reference implementation of the contracts primitive, including the 5 core tools and metrics surface. See the [Contracts plugin](components/plugins/contracts.md).
- **`paperclip-plugin-craft-dispatch` shipped.** Cross-company ticket bridge with flow-back on final transitions. See the [Craft Dispatch plugin](components/plugins/craft-dispatch.md).
- **V8.x of the contracts plugin landed.** Negotiation-backend hook surface (currently a stub) added so external counterparties can be plugged in without touching the core lifecycle.

## In flight

Work that is actively underway. No dates promised.

- **Cockpit-as-plugin.** Replacement for the archived Next.js dashboard, rebuilt as a plugin inside the governance company. What: same operator surface, hosted inside Paperclip. Why: removes a parallel service to maintain and pulls operator tooling into the same governance-owned plugin model as everything else. See [Cockpit](components/cockpit.md).
- **ADR-038 — Engineering Ticket Contract.** What: codify the implicit ticket-shape contract Nexus Engineering currently relies on. Why: referenced from several sources but not yet written; closing the loop unblocks downstream evals.
- **ADR-044 — Pluggable Negotiation Backend.** What: companion to ADR-043, specifies the interface that the Aurelius backend (and others) plug into for external contract negotiation. Why: the contracts plugin already exposes the hook surface; ADR-044 fills in the contract for what plugs in there.
- **Phase-1 → Phase-2 ramp.** What: raise `NEXUS_MAX_LIVE_WORKERS` past the current default of 1. Why: the global cap is a Phase-1 safety brake; lifting it is the next step in scaling concurrency, gated on the heartbeat's observability holding up at higher load.

## Planned / under consideration

Things on the planning surface, not yet committed. The cut between *in flight* and *planned* is intentional — anything below this line could be reshaped or dropped.

- **More craft companies.** What: QA, Research, and Editorial crafts in the spirit of the existing company taxonomy. Why: the two-class model only pays off when domains have multiple specialised crafts to dispatch to, not just Engineering.
- **Additional Aurelius flywheel rotations.** What: continue the reconstruction + simulator track work begun in the first rotations. Why: the flywheel only delivers compounding gains if rotations keep happening; this is the explicit commitment to keep turning it.
- **Run-quality eval expansion.** What: broaden the [eval registry](components/eval-registry.md) so the gap-scan has more dimensions to match postmortems against. Why: every uncovered failure class is a missed learning loop; expanding the registry shrinks that surface.
- **Webhook-driven dispatch supplement.** What: keep the heartbeat as the safety net but add webhook hooks for lower-latency hot-event dispatches across more plugins (currently only ACP uses this). Why: cuts time-to-spawn for events where the 5-minute heartbeat cadence is the bottleneck, without giving up the heartbeat's failure-tolerance guarantees.

---

This roadmap is curated, not exhaustive. If something is missing it may be in flight without being notable enough to call out — check the [Decisions Index](concepts/decisions-index.md) for the full ADR stream.

## See also

- [Decisions Index](concepts/decisions-index.md) — every ADR, grouped by theme
- [The Flywheel](architecture/flywheel.md) — the thesis the roadmap rolls up to
- [Layers Overview](architecture/layers-overview.md) — where each in-flight item lands
- [FAQ](faq.md) — answers to first-week questions
