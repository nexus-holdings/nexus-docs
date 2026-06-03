# Roadmap

<p class="lede">The full planning surface for Nexus — every feature that is shipped, actively in flight, proposed, or still exploratory, grouped by theme. Each entry records what it does, where it integrates, where the decision lives, what it depends on, and the next concrete step. This is the planning tool; the <a href="concepts/decisions-index.md">Decisions Index</a> is the formal record.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-06-03</span>
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
| [1 · Multi-company coordination](#1-multi-company-coordination) | 3 | — | 2 | 1 | Three company classes; first craft company is live; governance spawn pipeline is accepted (ADR-045) |
| [2 · Contracts](#2-contracts) | 1 | — | — | — | Inter-company agreements are a first-class, lifecycle-tracked primitive |
| [3 · Execution model & quality](#3-execution-model-quality) | 1 | — | 1 | — | Ticket flags are deterministic; formalising the implicit ticket contract is the remainder |
| [4 · Memory & retrieval](#4-memory-retrieval) | 1 | — | 3 | — | Promoter is live; retrieval quality is the active research frontier |
| [5 · Operator tooling](#5-operator-tooling) | 2 | 1 | 1 | — | Routine registration is live; Cockpit is being rebuilt as the agora governance plugin |
| [6 · Scaling & resilience](#6-scaling-resilience) | 1 | 1 | 2 | 1 | Concurrency is held at a Phase-1 brake, gated on observability |
| [7 · Learning loop & evals](#7-learning-loop-evals) | — | — | 1 | 1 | Every postmortem should mint an eval; expanding that surface |

---

## 1 · Multi-company coordination

How the holding company decomposes into domain companies (which own a problem space), craft companies (stateless execution), and the governance company that directs them — and how work crosses between them. See [Company classes](concepts/two-class-companies.md) for the concept.

### Company-class model

| | |
|---|---|
| **Status** | 🟢 Shipped |
| **Integrates** | [Company classes](concepts/two-class-companies.md), [Companies](concepts/companies.md) |
| **Source** | ADR-032, ADR-033 |
| **Depends on** | — |
| **Next step** | Finish migrating residual engineer roles out of domain companies into the Engineering craft (restructure Phases B–E) |

Three classes: domain companies (Aurelius, Lighthouse) own context and decompose goals into tickets; craft companies execute statelessly; the governance company (Nexus Holdings) directs the whole. This is the organising principle the rest of the substrate rolls up to. The model is accepted and live; the physical restructuring of older companies is the in-flight remainder.

### Company spawn pipeline (governance)

| | |
|---|---|
| **Status** | 🔵 Proposed |
| **Integrates** | Governance plugin ([agora](#governance-plugin-agora), Theme 5), [Craft Dispatch plugin](components/plugins/craft-dispatch.md), [Contracts plugin](components/plugins/contracts.md) |
| **Source** | ADR-045 (accepted) |
| **Depends on** | Company-class model; canonical provisioning (`provision_company.py`) |
| **Next step** | Finish the `create_child_company` stub against canonical provisioning; wire delegation through dispatch + contracts |

The codified path for the governance class to *grow the org chart*: a spec arrives, the C-suite drafts an implementation proposal, a human approves, and the governance company spawns child companies and delegates the initial work under contracts. ADR-045 is accepted; a working scaffold (agora) exists with the spec lifecycle and 21 passing tests, but the spawn step is still a stub. Sits above flat-dispatch — genesis, not routine routing.

### Cross-company dispatch

| | |
|---|---|
| **Status** | 🟢 Shipped |
| **Integrates** | [Craft Dispatch plugin](components/plugins/craft-dispatch.md) |
| **Source** | ADR-037 |
| **Depends on** | ADR-038 (Engineering Ticket Contract) for schema validation |
| **Next step** | Tighten the dispatched-ticket schema once ADR-038 lands |

The `craft_dispatch_ticket` tool lets a domain company file a well-formed ticket into a craft company with full context and a reverse link (`origin_kind`/`origin_id`), and receive status flow-back on completion. The plugin, the tool, and flow-back on final transitions (`flowback.js`) are all live; the only remaining hardening is schema validation, which lands once the ticket contract (ADR-038) is formalised.

### Nexus Engineering (craft company)

| | |
|---|---|
| **Status** | 🟢 Shipped |
| **Integrates** | [Craft Dispatch plugin](components/plugins/craft-dispatch.md), [Agent Catalog](components/agent-catalog.md) |
| **Source** | ADR-032, ADR-033 |
| **Depends on** | Company-class model |
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

The company-class model only compounds when domains have several specialised crafts to dispatch to. QA, Research, and Editorial are the named candidates. Directional only — no design started.

---

## 2 · Contracts

Inter-company agreements as an explicit, first-class primitive: scope, acceptance criteria, terms, and a lifecycle, tracked between companies on the substrate. See [Contracts](concepts/contracts.md).

### Contracts primitive

| | |
|---|---|
| **Status** | 🟢 Shipped |
| **Integrates** | [Contracts plugin](components/plugins/contracts.md), [Contracts concept](concepts/contracts.md) |
| **Source** | ADR-043 |
| **Depends on** | — |
| **Next step** | Wire the Cockpit contracts views (see Theme 5) |

Two-party agreements moved from implicit (a couple of columns on a dispatched ticket) to explicit: a `contracts` table with scope, acceptance criteria, terms, and a lifecycle (`draft → active → fulfilled`). The plugin ships the five core tools (`create_contract`, `list_contracts`, `link_issue_to_contract`, `update_contract_status`, `verify_acceptance_criterion`) plus a metrics surface.

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
| **Status** | 🟢 Shipped |
| **Integrates** | Paperclip issues (`execution_workspace_settings`), [ACP plugin](components/plugins/acp.md), skills |
| **Source** | ADR-042 |
| **Depends on** | — |
| **Next step** | Broaden the set of skills that read flags as new deterministic hand-offs are needed |

Structured JSON flags on a ticket make skill execution deterministic — e.g. a flag telling the inbox skill to pick up a specific ticket rather than re-running triage. The `skill_flags` helper, the read/write path (`nexus-core/nexus/execution/skill_flags.py`), and the inferred-flag path (`PAPERCLIP_TASK_ID`) are all live and covered by tests.

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
| **Next step** | Migrate the standalone Python worker onto the now-live catalog-defined routine form |

The dispatch loop that wakes each company, checks its inbox, and spawns agents on backlog tickets. Live as a systemd timer. The YAML-routine form is defined and routine registration now runs (below), so the remaining work is moving the heartbeat itself onto that path rather than the hard-coded worker.

### Governance plugin (agora)

| | |
|---|---|
| **Status** | 🟡 In flight |
| **Integrates** | Paperclip plugin surface, governance company, [nexus-mcp](components/nexus-mcp.md), [Contracts](components/plugins/contracts.md) + [Craft Dispatch](components/plugins/craft-dispatch.md) |
| **Source** | ADR-033 (Cockpit → Platform merge), ADR-045 (spawn pipeline) |
| **Depends on** | — |
| **Next step** | Adopt the agora scaffold on Linux; conform it to canonical primitives (drop REST shim, finish spawn against `provision_company.py`, delegate via dispatch/contracts); reach Cockpit parity on the operator views |

The standalone Cockpit is [archived](components/cockpit.md); its replacement is a plugin hosted inside the governance company. That plugin is **agora** — a working v0.2 scaffold (SDK manifest, spec lifecycle, 3 tools, 5 UI slots, 21 passing tests) that also implements the [spawn pipeline](#company-spawn-pipeline-governance) above. Same operator features, hosted in-platform, no parallel service to maintain. Two gaps remain: the operator views aren't yet at Cockpit parity (metrics, active companies, live transcript), and the `create_child_company` spawn step is still a stub.

### Cockpit contracts views

| | |
|---|---|
| **Status** | 🔵 Proposed |
| **Integrates** | [Contracts plugin](components/plugins/contracts.md), Governance plugin (agora) |
| **Source** | ADR-043 (Phase 1 build) |
| **Depends on** | Governance plugin (agora) |
| **Next step** | List + detail views: filter by company/status/backend; show scope, criteria, terms, linked issues, activity |

The contracts primitive has no operator UI yet. These views ride on the Governance plugin (agora) operator surface.

### Routine registration wiring

| | |
|---|---|
| **Status** | 🟢 Shipped |
| **Integrates** | Paperclip startup, [Routine Catalog](components/routine-catalog.md) |
| **Source** | Routine catalog design |
| **Depends on** | — |
| **Next step** | Migrate the heartbeat worker onto a catalog-defined routine now that registration is live |

`bootstrap_routines.py` reads `routine-catalog/routines/*.yaml` and registers each via `POST /api/companies/{id}/routines` — so `GET /api/companies/<id>/routines` returns the catalog-defined set rather than 0. The loop is closed; routines can now be catalog-defined rather than hard-coded.

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
- [Company classes](concepts/two-class-companies.md) — the organising principle behind Theme 1
- [FAQ](faq.md) — answers to first-week questions
