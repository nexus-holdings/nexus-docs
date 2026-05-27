# Roadmap

<p class="lede">The full planning surface for Nexus — every feature that is shipped, actively in flight, proposed, or still exploratory, grouped by theme. Each entry records what it does, where it integrates, where the decision lives, what it depends on, and the next concrete step. This is the planning tool; the <a href="concepts/decisions-index.md">Decisions Index</a> is the formal record.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-27</span>
  <span>Owner: Platform</span>
</div>

## How to read this

Every feature carries a status. The cut between stages is deliberate — anything below *In flight* can still be reshaped or dropped.

| Status | Meaning |
|---|---|
| 🟢 **Shipped** | Live and working today. Any follow-on hardening is noted under *Next step*. |
| 🟡 **In flight** | Actively being built; partially landed. |
| 🔵 **Proposed** | Designed — an ADR or design doc exists — but implementation hasn't started. |
| ⚪ **Exploratory** | Directional. The intent is set; no design committed yet. |

"Source" points at the ADR or design doc that owns the decision. "Integrates" links the component or concept page where the work lives. No dates are promised anywhere on this page; the ordering within each theme runs shipped → exploratory.

## At a glance

| Theme | 🟢 | 🟡 | 🔵 | ⚪ | Headline |
|---|:--:|:--:|:--:|:--:|---|
| [1 · Multi-company coordination](#1-multi-company-coordination) | 2 | 1 | 1 | 1 | Two-class model is the load-bearing structure; first craft company is live |
| [2 · Contracts & negotiation](#2-contracts-negotiation) | 1 | 1 | 1 | — | Contracts are a first-class primitive; external negotiation backend is the next unlock |
| [3 · Execution model & quality](#3-execution-model-quality) | — | — | 2 | — | Formalising the implicit ticket contract so dispatch is verifiable |
| [4 · Memory & retrieval](#4-memory-retrieval) | 1 | — | 3 | — | Promoter is live; retrieval quality is the active research frontier |
| [5 · Operator tooling](#5-operator-tooling) | 1 | 1 | 2 | — | Cockpit is being rebuilt as an in-platform plugin |
| [6 · Scaling & resilience](#6-scaling-resilience) | 1 | 1 | 2 | 1 | Concurrency is held at a Phase-1 brake, gated on observability |
| [7 · Learning loop & evals](#7-learning-loop-evals) | — | — | 1 | 1 | Every postmortem should mint an eval; expanding that surface |

---

## 1 · Multi-company coordination

How the holding company decomposes into domain companies (which own a problem space) and craft companies (stateless execution), and how work crosses between them. See [Two-class companies](concepts/two-class-companies.md) for the concept.

### Two-class company model

| | |
|---|---|
| **Status** | 🟢 Shipped |
| **Integrates** | [Two-class companies](concepts/two-class-companies.md), [Companies](concepts/companies.md) |
| **Source** | ADR-032, ADR-033 |
| **Depends on** | — |
| **Next step** | Finish migrating residual engineer roles out of domain companies into the Engineering craft (restructure Phases B–E) |

Domain companies (Aurelius, Lighthouse) own context and decompose goals into tickets; craft companies execute statelessly. This is the organising principle the rest of the substrate rolls up to. The model is accepted and live; the physical restructuring of older companies is the in-flight remainder.

### Cross-company dispatch

| | |
|---|---|
| **Status** | 🟢 Shipped |
| **Integrates** | [Craft Dispatch plugin](components/plugins/craft-dispatch.md) |
| **Source** | ADR-037 |
| **Depends on** | ADR-038 (Engineering Ticket Contract) for schema validation |
| **Next step** | Verify flow-back on final transitions; tighten the dispatched-ticket schema once ADR-038 lands |

The `craft_dispatch_ticket` tool lets a domain company file a well-formed ticket into a craft company with full context and a reverse link (`origin_kind`/`origin_id`), and receive status flow-back on completion. The plugin and tool are live; schema validation hardens once the ticket contract is formalised.

### Nexus Engineering (craft company)

| | |
|---|---|
| **Status** | 🟢 Shipped |
| **Integrates** | [Craft Dispatch plugin](components/plugins/craft-dispatch.md), [Agent Catalog](components/agent-catalog.md) |
| **Source** | ADR-032, ADR-033 |
| **Depends on** | Two-class model |
| **Next step** | Route more domain work through it as the restructure completes |

The first craft company — a stateless engineering execution engine. Receives spec or review tickets from domain companies, returns committed code. Provisioned and active.

### Nexus Observability (craft company)

| | |
|---|---|
| **Status** | 🔵 Proposed |
| **Integrates** | Holdings governance, cross-company dispatch |
| **Source** | ADR-033 (Phase 1b) |
| **Depends on** | Nexus Engineering pattern proven |
| **Next step** | Provision from the company template once the Engineering craft validates the dispatch loop |

A cross-cutting craft company that ensures every project has instrumentation, logging, alerting, and dashboards — working for the platform and all domain companies rather than being scoped to one. Specced, not yet provisioned.

### More craft companies (QA, Research, Editorial)

| | |
|---|---|
| **Status** | ⚪ Exploratory |
| **Integrates** | Cross-company dispatch model |
| **Source** | ADR-032 (future candidates) |
| **Depends on** | Multiple domain companies generating demand for each craft |
| **Next step** | Wait for dispatch volume to justify the first additional craft, then design it |

The two-class model only compounds when domains have several specialised crafts to dispatch to. QA, Research, and Editorial are the named candidates. Directional only — no design started.

---

## 2 · Contracts & negotiation

Inter-company agreements as an explicit, first-class primitive — and the path to plugging external counterparties into that primitive. See [Contracts](concepts/contracts.md).

### Contracts primitive

| | |
|---|---|
| **Status** | 🟢 Shipped |
| **Integrates** | [Contracts plugin](components/plugins/contracts.md), [Contracts concept](concepts/contracts.md) |
| **Source** | ADR-043 |
| **Depends on** | — |
| **Next step** | Wire the Cockpit contracts views (see Theme 5) and the external negotiation backend (below) |

Two-party agreements moved from implicit (a couple of columns on a dispatched ticket) to explicit: a `contracts` table with scope, acceptance criteria, terms, and a lifecycle (`draft → negotiating → active → fulfilled`). The plugin ships the five core tools (`create_contract`, `list_contracts`, `link_issue_to_contract`, `update_contract_status`, `verify_acceptance_criterion`) plus a metrics surface.

### Pluggable negotiation backend

| | |
|---|---|
| **Status** | 🔵 Proposed |
| **Integrates** | [Contracts plugin](components/plugins/contracts.md) hook surface, [Aurelius](https://aurelius.ai) |
| **Source** | ADR-044 (forthcoming, companion to ADR-043) |
| **Depends on** | Contracts primitive (shipped) |
| **Next step** | Draft ADR-044 specifying the backend interface; replace the stub hook with the Aurelius HTTP integration |

When a contract enters `negotiating` with an external counterparty, the negotiation is delegated to a pluggable backend (Aurelius is the first). The contracts plugin already exposes the hook surface as a stub; ADR-044 fills in the interface contract, and the Aurelius backend plugs in behind it.

### Aurelius flywheel rotations

| | |
|---|---|
| **Status** | 🟡 In flight |
| **Integrates** | [The Flywheel](architecture/flywheel.md), contracts as a training-signal source |
| **Source** | Flywheel architecture (reconstruction + simulator tracks) |
| **Depends on** | Contracts primitive (now shipped — unblocks the real-anchored signal) |
| **Next step** | Continue the reconstruction and self-play simulator tracks; feed live contract transcripts into the training pool |

Each rotation reconstructs counterparty intent from real negotiation rounds, trains a self-play simulator on it, and promotes the best variants to production — three arms all emitting training signal into one GRPO pool. The early tracks landed; the reconstruction and simulator tracks are the active rotation, newly unblocked now that contracts give a real-anchored signal source.

---

## 3 · Execution model & quality

Making ticket-driven execution legible and verifiable, so dispatched work succeeds deterministically rather than relying on convention.

### Engineering ticket contract

| | |
|---|---|
| **Status** | 🔵 Proposed |
| **Integrates** | [Craft Dispatch plugin](components/plugins/craft-dispatch.md), Nexus Engineering |
| **Source** | ADR-038 (referenced from ADR-037, not yet written) |
| **Depends on** | — |
| **Next step** | Draft ADR-038 to codify the acceptance-criteria format, context shape, and priority enum a craft company can rely on |

Nexus Engineering currently relies on an *implicit* contract for what a well-formed inbound ticket looks like. ADR-038 will make that explicit so dispatched tickets can be schema-validated — which in turn unblocks downstream evals against a known shape. The schema is implicit in the dispatch tool's `spec` parameter today; the standalone record is missing.

### Ticket-flag protocol

| | |
|---|---|
| **Status** | 🔵 Proposed |
| **Integrates** | Paperclip issues (`execution_workspace_settings`), [ACP plugin](components/plugins/acp.md), skills |
| **Source** | ADR-042 |
| **Depends on** | — |
| **Next step** | Implement the `skill_flags` helper and read path; the inferred-flag path (`PAPERCLIP_TASK_ID`) already exists |

Structured JSON flags on a ticket make skill execution deterministic — e.g. a flag telling the inbox skill to pick up a specific ticket rather than re-running triage. The design is locked; the explicit-flag read path is the implementation gap.

---

## 4 · Memory & retrieval

The memory layer is live; the open work is almost entirely about retrieval *quality*. See [Nexus Memory](components/nexus-memory.md) and [Memory layer](architecture/memory-layer.md).

### MemPalace → Context-1 promoter

| | |
|---|---|
| **Status** | 🟢 Shipped |
| **Integrates** | [Nexus Memory](components/nexus-memory.md), ChromaDB (`:8101`) |
| **Source** | Memory architecture |
| **Depends on** | — |
| **Next step** | Layer hybrid retrieval and query expansion (below) on top of the existing dense index |

A background timer (every 15 min) embeds newly-promoted drawers with BGE-M3 and upserts them into Context-1. Live and autonomous.

### Hybrid retrieval (BM25 + dense)

| | |
|---|---|
| **Status** | 🔵 Proposed |
| **Integrates** | Context-1 retriever, ChromaDB indexing |
| **Source** | Memory retrieval roadmap (priority 1) |
| **Depends on** | Promoter (shipped) |
| **Next step** | Index BGE-M3's native sparse vectors alongside dense; merge results at query time via reciprocal rank fusion |

Pure dense retrieval misses exact-term matches. BGE-M3 already emits a sparse vector; using it gives keyword recall without a second model. Highest-value retrieval item, ~6–10h of work.

### Domain-tuned embedder

| | |
|---|---|
| **Status** | 🔵 Proposed |
| **Integrates** | Context-1 retriever, clause/contract corpus in MemPalace |
| **Source** | Memory retrieval roadmap (priority 3) + clause-embedding pilot spec |
| **Depends on** | Hybrid retrieval (to isolate the embedder's contribution) |
| **Next step** | Fine-tune BGE-M3 on contract + session data; target top-1 match 0.55–0.70 → 0.75–0.85 on contract content |

The general embedder underperforms on dense legal/contract text. The benchmark harness is ready and the training corpus exists (30K+ clause drawers); the training code does not. Open question still to resolve: contracts-only vs. mixed-corpus training.

### Query expansion

| | |
|---|---|
| **Status** | 🔵 Proposed |
| **Integrates** | Context-1 retriever (`/v1/retrieve`, behind an `expand` flag) |
| **Source** | Memory retrieval roadmap (priority 2) |
| **Depends on** | — |
| **Next step** | Add a one-shot Haiku rewrite of question-shaped queries into keyword-rich variants before embedding |

Lowest-effort retrieval win (~2–3h) for the weak, question-shaped queries in the current benchmark. Gated behind a flag because it adds noise for already keyword-shaped queries.

---

## 5 · Operator tooling

The operator surface and the recurring-task machinery that keeps companies ticking. See [Cockpit](components/cockpit.md) and [Routine Catalog](components/routine-catalog.md).

### Company heartbeat

| | |
|---|---|
| **Status** | 🟢 Shipped |
| **Integrates** | [Nexus Core](components/nexus-core.md), [Heartbeat](concepts/heartbeat.md) |
| **Source** | Execution-layer DFAs |
| **Depends on** | — |
| **Next step** | Move from the standalone Python worker to the catalog-defined routine once registration wiring lands (below) |

The dispatch loop that wakes each company, checks its inbox, and spawns agents on backlog tickets. Live as a systemd timer; the YAML-routine form is defined and the wiring to auto-register it is the remaining gap.

### Cockpit-as-plugin

| | |
|---|---|
| **Status** | 🟡 In flight |
| **Integrates** | Paperclip plugin surface, governance company, [nexus-mcp](components/nexus-mcp.md) |
| **Source** | ADR-033 (Cockpit → Platform merge) |
| **Depends on** | — |
| **Next step** | Rebuild the operator surface (metrics, active companies, live transcript) as a governance-hosted plugin and retire the archived Next.js app |

The standalone Cockpit is [archived](components/cockpit.md). Its replacement is being built as a plugin inside the governance company — same operator features, hosted in-platform, no parallel service to maintain. The live-transcript view is the operationally-visible gap until it lands.

### Cockpit contracts views

| | |
|---|---|
| **Status** | 🔵 Proposed |
| **Integrates** | [Contracts plugin](components/plugins/contracts.md), Cockpit-as-plugin |
| **Source** | ADR-043 (Phase 1 build) |
| **Depends on** | Cockpit-as-plugin |
| **Next step** | List + detail views: filter by company/status/backend; show scope, criteria, terms, linked issues, activity |

The contracts primitive has no operator UI yet. These views ride on the Cockpit-as-plugin rebuild.

### Routine registration wiring

| | |
|---|---|
| **Status** | 🔵 Proposed |
| **Integrates** | Paperclip startup, [Routine Catalog](components/routine-catalog.md) |
| **Source** | Routine catalog design |
| **Depends on** | — |
| **Next step** | On startup, read `routine-catalog/routines/*.yaml` and register each via `POST /api/companies/{id}/routines` |

The routine YAMLs sit on disk but aren't auto-registered — `GET /api/companies/<id>/routines` currently returns 0. A small startup hook closes the loop and lets the heartbeat (and other routines) be catalog-defined rather than hard-coded.

---

## 6 · Scaling & resilience

Capacity, cost, and the safety brakes around running more agents concurrently. Several items here are operational ADRs rather than user-facing features.

### Prompt-cache optimization

| | |
|---|---|
| **Status** | 🟡 In flight |
| **Integrates** | Agent dispatch, Meridian (`:3456`) |
| **Source** | ADR-028 |
| **Depends on** | — |
| **Next step** | Move from measurement to enforcement — switch multi-turn sessions to the 1-hour cache TTL by default |

Measurement infrastructure for prompt-cache hit rates is built. Turning the findings into a default (1-hour TTL on multi-turn sessions) is the remaining step.

### Session-limit mitigations

| | |
|---|---|
| **Status** | 🟢 Shipped |
| **Integrates** | Agent dispatch |
| **Source** | ADR-029 (tracking) |
| **Depends on** | — |
| **Next step** | Maintain mitigations and weekly tracking until the upstream limit behaviour is fixed |

Mitigations for upstream session-limit behaviour are in place and tracked. Ongoing maintenance rather than new build.

### Worker-concurrency ramp (Phase 1 → 2)

| | |
|---|---|
| **Status** | 🔵 Proposed |
| **Integrates** | [Nexus Core](components/nexus-core.md) scheduler, `NEXUS_MAX_LIVE_WORKERS` ([env vars](reference/env-vars.md)) |
| **Source** | Operational policy |
| **Depends on** | Heartbeat observability holding up at higher load |
| **Next step** | Raise `NEXUS_MAX_LIVE_WORKERS` past the current default of 1 once observability is trusted at load |

The global worker cap is a deliberate Phase-1 safety brake. Lifting it is the next concurrency step — gated, not dated, on the observability story being solid enough to trust unattended.

### Direct-API fallback

| | |
|---|---|
| **Status** | 🔵 Proposed |
| **Integrates** | Meridian (`:3456`), token-heavy analysis phases |
| **Source** | ADR-030 (evaluation complete) |
| **Depends on** | — |
| **Next step** | Implement a narrow API fallback for analysis-only phases as a proof-of-concept (P3) |

Evaluation concluded: stay on the subscription path, but add a narrow direct-API fallback for the most token-heavy analysis phases. Scoped small; not started.

### Webhook-driven dispatch supplement

| | |
|---|---|
| **Status** | ⚪ Exploratory |
| **Integrates** | Paperclip webhooks, plugin hooks (only [ACP](components/plugins/acp.md) uses this today) |
| **Source** | Planning surface |
| **Depends on** | — |
| **Next step** | Prototype webhook hot-event dispatch on one more plugin, keeping the heartbeat as the safety net |

The 5-minute heartbeat is the failure-tolerant baseline. Webhooks would cut time-to-spawn for hot events without giving up that guarantee — additive, not a replacement. Directional.

---

## 7 · Learning loop & evals

Turning failures into durable, reusable checks — the governance loop that makes the substrate improve rather than just run. See [Governance](architecture/governance.md) and [Eval Registry](components/eval-registry.md).

### Run-quality eval expansion

| | |
|---|---|
| **Status** | 🔵 Proposed |
| **Integrates** | [Eval Registry](components/eval-registry.md), postmortem → ADR → eval loop |
| **Source** | Governance loop |
| **Depends on** | — |
| **Next step** | Add eval dimensions for each newly-postmortemed failure class so the gap-scan has more to match against |

The `run-quality` eval exists; the work is broadening it. Every uncovered failure class is a missed learning loop, so each postmortem should mint a matching eval. This is the concrete expansion of that principle.

### Verification-skill family

| | |
|---|---|
| **Status** | ⚪ Exploratory |
| **Integrates** | [Skills Catalog](components/skills-catalog.md), Eval Registry |
| **Source** | Platform design research |
| **Depends on** | Run-quality eval expansion |
| **Next step** | Add `verify-*` skills as failure modes get postmortems, replacing instruction-only prompts with explicit rubric checks |

Research finding: explicit verification skills outperform instruction-only prompts by 4–40 points on quality metrics. The catalog should grow a `verify-*` family as the eval surface expands. Directional, paced by postmortem volume.

---

This roadmap is curated from the ADR stream, the design docs, and the live deployment — it reflects what is actually running, not just what was decided. If something here drifts from reality, the deployment is the source of truth; flag it and the entry gets corrected.

## See also

- [Decisions Index](concepts/decisions-index.md) — every ADR, grouped by theme; the formal record behind these entries
- [The Flywheel](architecture/flywheel.md) — the thesis the roadmap rolls up to
- [Layers Overview](architecture/layers-overview.md) — where each item lands across the four layers
- [Two-class companies](concepts/two-class-companies.md) — the organising principle behind Theme 1
- [FAQ](faq.md) — answers to first-week questions
