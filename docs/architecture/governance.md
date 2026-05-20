# Governance

<p class="lede">Governance is the layer that decides <strong>whether work was good</strong>. Evals score outcomes, postmortems autopsy failures, ADRs record what the substrate decided and why, contracts define what one company owes another. Without governance, Nexus produces work but can't tell whether it's improving.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-19</span>
  <span>Owner: Platform</span>
</div>

## What it does

The governance layer turns observed work into structured judgments. Four primitives:

| Primitive | Captures | Triggered by |
|---|---|---|
| **Eval** | A scoring rubric applied to a class of work | Routine runs, post-merge checks, ticket gates |
| **Postmortem** | A failure record with root cause and follow-ups | An "interesting failure" — repeat reject, merge conflict, zombie session |
| **ADR** | An architectural decision recorded with context | A decision made (often emerging from a postmortem) |
| **Contract** | A formal agreement between two companies | A craft company hired by a domain company |

These four primitives are the durable artifacts of governance. Everything else (skill flags, circuit breakers, model selection rules) reads from them.

## The feedback shape

<img src="../../assets/diagrams/governance-cycle.svg" alt="Governance learning cycle — Work happens → Eval scores it → ADR codifies the decision → Future work is gated, in a left-to-right row. Postmortem sits below as a branch from Work on failure, feeding back into ADR. A brass 'future work improves' loop returns from the right endpoint to the left.">


The governance loop closes when an ADR (or an updated eval) changes how future work is routed. That's where the substrate "learns" — not via online weight updates, but via a corpus of decisions that subsequent work has to respect.

## Evals

An eval is a **versioned scoring rubric** for a specific class of work. Each eval has:

- A **name** (`code-quality`, `security-scan`, `design-adherence`, `review-efficiency`)
- A **version** (patch / minor / major, semver-style)
- A **scoring function** — sometimes a deterministic check, sometimes an LLM-graded rubric
- A **target** — what kind of work this eval applies to (e.g., "any PR merged to main", "any review verdict")

Evals live in the [Eval Registry](../components/eval-registry.md), which is a git-tracked YAML store. Changes are PR-gated and require evidence in the changelog ("this version moved the average score from X to Y on the existing corpus").

When the substrate scores a piece of work, it records `(eval_id, eval_version, score, evidence)` to the metrics DB. Versioning is critical: a score is meaningless without the version that produced it.

## Postmortems

A postmortem is generated when the execution layer detects an "interesting failure":

- Repeated review rejects on the same ticket
- Merge conflict after an approved review
- Zombie session (worker exited without transitioning state)
- Circuit breaker trip (N consecutive failures for an agent)
- Cross-arm signal mismatch (an eval that contradicts another)

The pipeline (DFA 20 in `docs/state-machines.md`):

1. **Detect** — an event fires from the execution layer
2. **Build** — assemble the record: ticket, session transcript, diffs, agent output, eval scores
3. **Scan** — query the eval registry for evals that *should* have caught this; surface gaps
4. **Write** — produce a postmortem document (often via a postmortem agent)
5. **File** — create a follow-up ticket if the postmortem identifies a fix

Postmortems live in `docs/postmortems/<id>.md` in the relevant repo. They are *never deleted* — even superseded ones are kept as history.

## ADRs

An ADR — Architecture Decision Record — captures a *decision* the substrate (or its operators) made, with enough context that a future reader can understand why.

Standard shape:

```markdown
# ADR-032: Two-class company model (craft + domain)

Status: accepted
Date: 2026-03-12

## Context
Companies had been ad-hoc — each had its own roster of agents, its own
process for triage. Onboarding a new company required ~2 weeks of
configuration work.

## Decision
Split companies into two formal classes:
  - Domain companies: own context, hold the relationship with the customer
  - Craft companies: stateless executors, hired by domain companies

## Consequences
- Onboarding a craft company is ~1 day (template + skill flags)
- Cross-company contracts are now first-class (ADR-043 follows from this)
- Existing companies needed migration — see migration-032.md
```

ADRs are numbered sequentially across the org. The full set lives in `docs/decisions/` (currently ~15 ADRs).

Reading the ADR set is how a new operator gets up to speed on *why Nexus is shaped this way*. The code shows what; the ADRs show why.

## Contracts

A contract is a structured agreement between two companies inside the substrate. Most commonly:

- A **domain company** (with the customer relationship) needs work done
- It **hires a craft company** for a specific scope (build a feature, run an audit)
- The contract defines: scope, acceptance criteria, deadline, exit conditions

Contracts go through a lifecycle:

```
draft → negotiating → active → fulfilled
                              ↘ cancelled
```

The Contracts plugin (`paperclip-plugin-contracts`) implements this lifecycle. External-party contracts (with humans / other firms) route through a pluggable negotiation backend.

See [ADR-043: Contracts as first-class primitives](../concepts/decisions-index.md) for the design rationale.

## Why governance has to live at the substrate level

A tempting alternative: let each flywheel govern itself. Each vertical defines its own evals, writes its own postmortems, makes its own decisions.

Why that fails: **the substrate is the only place where multiple flywheels meet**. A skill flag set by one flywheel needs to apply to agents that serve another. An eval that catches a regression in vertical A should warn vertical B. A contract from craft company X to domain company Y can't be enforced if governance is duplicated per flywheel.

Centralizing governance at the substrate level is what makes the substrate a *substrate* and not just a shared library. The cost is that governance has to be well-engineered. The benefit is that improvements compound across every flywheel.

## What it does not do

The governance layer **does not**:

- **Track entity state** — that's the [platform layer](platform-layer.md). Governance reads platform state and writes judgments *about* it.
- **Run the agents that produce evals or postmortems** — those are tickets in the substrate, dispatched by the [execution layer](execution-layer.md). Governance defines *what* a postmortem looks like; execution *makes* it happen.
- **Store the artifacts** — eval records, postmortems, ADRs are stored in the [memory layer](memory-layer.md) (for retrieval) and in repos (for version control). Governance is the *shape* of these artifacts, not the storage.

The governance layer is small on purpose — a few schemas, a few state machines, a few code paths that update them. The richness lives in the artifacts themselves.

## Implementation

Governance is the most distributed of the four layers. Components involved:

- **[Eval Registry](../components/eval-registry.md)** — versioned YAML specs + rubrics
- **[Contracts plugin](../components/plugins/contracts.md)** — contract lifecycle
- **Postmortem pipeline** — code in [Nexus Core](../components/nexus-core.md) (`postmortem.py`), driven by DFA 20
- **ADR set** — markdown files in `docs/decisions/` per repo, indexed via the [Decisions Index](../concepts/decisions-index.md)

There is no single "governance daemon." The layer is a *contract* enforced by the components above.

## See also

- [Eval Registry](../components/eval-registry.md) — versioned scoring specs
- [Postmortems](../concepts/postmortems.md) — when and how
- [Decisions Index](../concepts/decisions-index.md) — the ADR catalog
- [Contracts plugin](../components/plugins/contracts.md) — inter-company agreements
- [Platform Layer](platform-layer.md) — what governance reads from
- [Memory Layer](memory-layer.md) — where artifacts are stored for retrieval
- [Execution Layer](execution-layer.md) — what triggers postmortems
