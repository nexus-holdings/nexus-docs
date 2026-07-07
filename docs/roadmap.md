# Roadmap

<p class="lede">The full planning surface for Nexus — every feature that is shipped, actively in flight, proposed, or still exploratory, grouped by theme. Each entry records what it does, where it integrates, where the decision lives, what it depends on, and the next concrete step. This is the planning tool; the <a href="concepts/decisions-index.md">Decisions Index</a> is the formal record.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-07-07</span>
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
| [1 · Multi-company coordination](#1-multi-company-coordination) | 4 | — | 1 | 2 | Three company classes with a shared class register; the governance spawn pipeline is live end-to-end (ADR-045/046) |
| [2 · Contracts](#2-contracts) | 1 | — | — | — | Inter-company agreements are a first-class, lifecycle-tracked primitive |
| [3 · Execution model & quality](#3-execution-model-quality) | 2 | — | — | — | The ticket contract is written (ADR-038) and completion is mechanically verified at the merge HEAD |
| [4 · Memory & retrieval](#4-memory-retrieval) | 1 | — | 3 | — | Promoter is live; retrieval quality is the active research frontier |
| [5 · Operator tooling](#5-operator-tooling) | 3 | 1 | 1 | — | The agora governance console is installed (specs, children, contracts); operator-view parity with the old Cockpit is the remainder |
| [6 · Scaling & resilience](#6-scaling-resilience) | 2 | 1 | 2 | 1 | Guardrails are live and battle-tested; the first autonomous ticket merged verified on pilot night |
| [7 · Learning loop & evals](#7-learning-loop-evals) | 1 | — | 1 | — | Every postmortem should mint an eval; expanding that surface |

---

## 1 · Multi-company coordination

How the holding company decomposes into domain companies (which own a problem space), craft companies (stateless execution), and the governance company that directs them — and how work crosses between them. See [Company classes](concepts/two-class-companies.md) for the concept.

### Company-class model

| | |
|---|---|
| **Status** | 🟢 Shipped |
| **Integrates** | [Company classes](concepts/two-class-companies.md), [Companies](concepts/companies.md) |
| **Source** | ADR-032, ADR-033, ADR-046 |
| **Depends on** | — |
| **Next step** | Finish migrating residual engineer roles out of domain companies into the Engineering craft (restructure Phases B–E) |

Three classes: domain companies (Aurelius, Lighthouse) own context and decompose goals into tickets; craft companies execute statelessly; the governance company (Nexus Holdings) directs the whole. This is the organising principle the rest of the substrate rolls up to. Class and lineage are recorded in a shared [class register](concepts/two-class-companies.md#where-class-lives) (ADR-046) written at company birth and readable platform-wide. The model is accepted and live; the physical restructuring of older companies is the in-flight remainder.

### Company spawn pipeline (governance)

| | |
|---|---|
| **Status** | 🟢 Shipped |
| **Integrates** | Governance plugin ([agora](#governance-plugin-agora), Theme 5), [Craft Dispatch plugin](components/plugins/craft-dispatch.md), [Contracts plugin](components/plugins/contracts.md) |
| **Source** | ADR-045, ADR-046 |
| **Depends on** | Company-class model; canonical provisioning (`provision_company.py`) |
| **Next step** | Governance oversight of descendants' contracts; guardrails (budgets, sprawl monitoring) before any autonomous operation |

The codified path for the governance class to *grow the org chart*, now live end-to-end: a spec arrives, the C-suite drafts an implementation proposal, a **human approves** (the gate is mandatory), and the governance company spawns the child through canonical provisioning and delegates the initial work under contracts. The spawn is idempotent on the spec — replays can never double-create — and the spec that justified the company also defines it: its class and lineage land in the class register, its **agents carry the spec as their mandate**, and its **first project is the spec's work**, with the repo bound as the workspace. Sits above flat-dispatch — genesis, not routine routing.

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
| **Status** | 🟢 Shipped |
| **Integrates** | Holdings governance, cross-company dispatch |
| **Source** | ADR-033 (Phase 1b) |
| **Depends on** | Nexus Engineering pattern proven |
| **Next step** | First dispatched build: session & spend dashboard + runaway-execution alert (initial issue filed under NEXHOL-NEXOBS-0001) |

A cross-cutting craft company that ensures every project has instrumentation, logging, alerting, and dashboards — working for the platform and all domain companies rather than being scoped to one. Specced, not yet provisioned.

Provisioned 2026-06-05 through the live spawn pipeline — the first company born with a spec-declared weighted goals section (w5 observability, w4 runaway alerts, w2 minimal overhead). Delegation contract NEXHOL-NEXOBS-0001 is goal-bound on both sides.

### More craft companies (QA, Research, Editorial)

| | |
|---|---|
| **Status** | ⚪ Exploratory |
| **Integrates** | Cross-company dispatch model |
| **Source** | ADR-032 (future candidates) |
| **Depends on** | Multiple domain companies generating demand for each craft |
| **Next step** | Wait for dispatch volume to justify the first additional craft, then design it |

The company-class model only compounds when domains have several specialised crafts to dispatch to. QA, Research, and Editorial are the named candidates. Directional only — no design started.

### Goal-aware coordination

| | |
|---|---|
| **Status** | 🟢 Shipped |
| **Integrates** | [Contracts](concepts/contracts.md), cross-company dispatch, [Governance layer](architecture/governance.md) |
| **Source** | ADR-047 (accepted) |
| **Depends on** | Contracts primitive (shipped), goal-weight register (shipped) |
| **Next step** | Live use: watch the paradox-watch and inflation flags as contracts accumulate; weight_revision approval flow when first needed |

When two companies work under a contract, each plans with the *other's* objectives in view — instead of ping-ponging, each optimising the same point in opposite directions. v1 shipped: every company is born with spec-derived **weighted goals** (1–5 ordinal leeway rubric: lower flexes first, weights set by the human-gated spec, extremes governance-gated thereafter); contracts bind both sides' goals; a `get_coordination_context` tool gives agents the read-before-you-act planning view. A conflict between two weight-5 goals with no trade-off space is a *paradox* — never agent-resolved, always escalated to the governance ancestor as a logged issue. Recurring escalations on one goal flag the goal itself for refinement. v2 shipped too: the governance console's **Goal Review** tab audits weight distributions across the org (inflation, unweighted, goalless flags) and reviews every open contract's goal pair, flagging 5-vs-5 pairs as paradox watch.

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
| **Status** | 🟢 Shipped |
| **Integrates** | [Craft Dispatch plugin](components/plugins/craft-dispatch.md), Nexus Engineering |
| **Source** | ADR-038 (Accepted; §5 amended 2026-06-09 with the verify gate) |
| **Depends on** | — |
| **Next step** | Hold the shape: changes are ADR-level acts |

The five-section ticket shape is the written contract every dispatch conforms to. Its completion protocol now ends in a mechanical **verify gate**: the merge-agent runs the ticket's test command (or one discovered from the repo's shape) at the merge HEAD before any merge stands — tests fail and the merge is undone, the ticket rolled back. Completion is verified, not claimed. See [Execution Guardrails](concepts/execution-guardrails.md).

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
| **Source** | ADR-033 (Cockpit → Platform merge), ADR-045 (spawn pipeline), ADR-046 (class register) |
| **Depends on** | — |
| **Next step** | Reach Cockpit parity on the operator views (metrics, active companies, live transcript) — descendants'-contracts oversight and the Goal Review audit are live |

The standalone Cockpit is [archived](components/cockpit.md); its replacement is **[agora](components/plugins/agora.md)** — installed and live in the governance company. It implements the full [spawn pipeline](#company-spawn-pipeline-governance): four agent tools (`list_specs`, `propose_implementation`, `create_child_company`, `delegate_spec`), the spec lifecycle with audit trail, and a governance console (specs with approve/reject, children with class + lineage detail, contracts with criteria checklists). Class and lineage read from the shared class register; contracts compose through the Contracts plugin's own tools rather than reimplementing them. The remaining gap is operator-view parity with the old Cockpit.

### Cockpit contracts views

| | |
|---|---|
| **Status** | 🟢 Shipped |
| **Integrates** | [Contracts plugin](components/plugins/contracts.md), Governance plugin ([agora](components/plugins/agora.md)) |
| **Source** | ADR-043 (Phase 1 build) |
| **Depends on** | Governance plugin (agora) |
| **Next step** | Governance oversight: surface *descendants'* contracts in the governance console (today each console lists only contracts where the viewing company is a party) |

The agora governance console lists a company's contracts with expandable detail: parties, scope, acceptance criteria as a checklist with verification timestamps, linked issues with their roles, and lifecycle dates. The known gap is visibility of contracts *between* child companies — the governance company is not a party to those, so they don't appear in its view yet.

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

### Attention budget & review queue

| | |
|---|---|
| **Status** | 🔵 Proposed |
| **Integrates** | [Operator Attention](concepts/operator-attention.md), [Cockpit](components/cockpit.md) / agora console |
| **Source** | [Operator Attention](concepts/operator-attention.md) (design page, 2026-07-07) |
| **Depends on** | Agora governance console |
| **Next step** | Classify every existing human-facing escalation into Page / Window / Digest / Archive; add queue-depth, item-age, and reversal-rate metrics to the console |

Operator attention is the only execution resource the substrate does not meter. This entry makes it one: interrupt classes with latency contracts, batched review windows, decision SLAs with fail-closed defaults, and recurrence-as-signal applied to checkpoints themselves. See the concept page for the full design.

---

## 6 · Scaling & resilience

Capacity, cost, and the safety brakes around running more agents concurrently. Several items here are operational ADRs rather than user-facing features.

### Prompt-cache optimization

| | |
|---|---|
| **Status** | 🟢 Shipped |
| **Integrates** | Agent dispatch, Meridian (`:3456`) |
| **Source** | ADR-028 |
| **Depends on** | — |
| **Next step** | Verify hit-rate improvement from performance records once execution re-enables |

Measurement infrastructure for prompt-cache hit rates is built. Turning the findings into a default (1-hour TTL on multi-turn sessions) is the remaining step.

Enforcement landed 2026-06-05: both spawn paths (nexus-core team spawner, ACP sessions) inject the extended-cache-ttl beta via ANTHROPIC_CUSTOM_HEADERS; default strategy is now 1hr per the ADR-028 break-even analysis.

### Session-limit mitigations

| | |
|---|---|
| **Status** | 🟢 Shipped |
| **Integrates** | Agent dispatch |
| **Source** | ADR-029 (tracking) |
| **Depends on** | — |
| **Next step** | Maintain mitigations and weekly tracking until the upstream limit behaviour is fixed |

Mitigations for upstream session-limit behaviour are in place and tracked. Ongoing maintenance rather than new build.

### Execution guardrails + first unsupervised pilot

| | |
|---|---|
| **Status** | 🟢 Shipped (battle-tested) |
| **Integrates** | [ACP plugin](components/plugins/acp.md), native run path, [Nexus Core](components/nexus-core.md) heartbeat |
| **Source** | ADR-048, ADR-049 (conditions discharged 2026-06-09) |
| **Depends on** | — |
| **Next step** | The bounded ramp: overnight windows → 24h → a full unsupervised week, postmortems driving fixes |

Five layered brakes between a spawn attempt and a runaway loop: a finished-work check anchored in a platform-owned merged registry, dedup/cooldown, a per-company circuit breaker, durable daily volume caps shared by every execution path, and the verify gate at completion. Proven the hard way on the first unsupervised pilot night (2026-06-09): a novel host-side retry loop created ~20 runs against a completed ticket and the guards cancelled every one before execution — the postmortem fixes (merged anchor, adapter defaults, two upstream bug reports) shipped the same evening. See [Execution Guardrails](concepts/execution-guardrails.md).

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
| **Status** | 🟢 Shipped |
| **Integrates** | [Eval Registry](components/eval-registry.md), postmortem → ADR → eval loop |
| **Source** | Governance loop |
| **Depends on** | — |
| **Next step** | Keep minting dimensions per postmortem; calibrate the three new dimensions on the first scored runs |

The `run-quality` eval exists; the work is broadening it. Every uncovered failure class is a missed learning loop, so each postmortem should mint a matching eval. This is the concrete expansion of that principle.

v1.1.0 shipped 2026-06-05 — three dimensions minted from the cascade postmortems: coordination_adherence (ADR-047 protocol), loop_discipline (one ticket, one execution arc), config_health (valid tested config).

### Verification-skill family

| | |
|---|---|
| **Status** | 🔵 Proposed |
| **Integrates** | [Skills Catalog](components/skills-catalog.md), Eval Registry |
| **Source** | Platform design research (4–40 pt rubric-vs-instruction finding); ADR to be written as the first step |
| **Depends on** | Run-quality eval expansion |
| **Next step** | Write the ADR, then seed the family from the three v1.1.0 dimensions — `verify-coordination`, `verify-loop-discipline`, `verify-config` — replacing their instruction-only prompts with explicit rubric checks |

Research finding: explicit verification skills outperform instruction-only prompts by 4–40 points on quality metrics. Promoted from exploratory 2026-07-07: the eval layer is the acknowledged weak point of the governance loop, and this family is its highest-leverage expansion. Each new postmortem-minted eval dimension should ship with its `verify-*` skill rather than trailing it.

---

This roadmap is curated from the ADR stream, the design docs, and the live deployment — it reflects what is actually running, not just what was decided. If something here drifts from reality, the deployment is the source of truth; flag it and the entry gets corrected.

## See also

- [Decisions Index](concepts/decisions-index.md) — every ADR, grouped by theme; the formal record behind these entries
- [The Flywheel](architecture/flywheel.md) — the thesis the roadmap rolls up to
- [Layers Overview](architecture/layers-overview.md) — where each item lands across the four layers
- [Company classes](concepts/two-class-companies.md) — the organising principle behind Theme 1
- [FAQ](faq.md) — answers to first-week questions
