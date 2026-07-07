# Operator Attention

<p class="lede">Compute, tokens, and spend are all budgeted and capped. The operator's attention is not — and it is the scarcest resource in the system. Every safety mechanism Nexus has ultimately routes a decision to a human; an unbudgeted human queue fails exactly like an unbudgeted spawn loop: it compounds.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> proposed</span>
  <span>Updated 2026-07-07</span>
  <span>Owner: Platform</span>
</div>

## Why this exists

Trace where the substrate's guarantees come from and they all end at the same place. The [spawn pipeline](../architecture/governance.md) requires human approval. [Goal paradoxes](goal-aware-coordination.md) escalate to a human. Extreme goal-weight changes pass a human gate. [Postmortems](postmortems.md) file tickets a human triages. Unverified merges are "loudly flagged" — at a human. Idle-escalation pings a human. The [guardrails](execution-guardrails.md) bound what agents can spend; nothing bounds what the system can ask of its operator.

At today's scale this is invisible: the queue is short and the operator is the platform's author. But the failure mode is already legible, because it is the platform's own founding failure mode wearing a different coat. Unattended loops were contained by making failure *non-compounding*. An overloaded review queue compounds the same way:

1. Escalations arrive faster than decisions leave.
2. The operator batches, defers, or rubber-stamps — vigilance degrades under load, silently.
3. Deferred decisions block tickets, which triggers idle escalations, retries, and postmortems — **which themselves join the queue**.

Step 3 is the loop. The queue generates queue. A platform whose thesis is "autonomous execution is allowed to fail — it is not allowed to compound" has to extend that thesis to the one component it cannot restart: the human.

There is a second, quieter reason. Rubber-stamping under load doesn't just risk a bad approval — it *poisons the learning loop*. The governance layer treats human decisions as ground truth: approvals gate spawns, paradox resolutions reshape goals, postmortem triage decides what becomes an eval. A fatigued decision enters the corpus with the same weight as a considered one. Attention quality is upstream of everything the substrate learns.

## The principle

**Treat operator attention as a metered, budgeted execution resource — the same discipline applied to workers, spend, and sessions.** Concretely:

- Attention has a **daily budget** (decisions per day, minutes per day — pick a unit and count it).
- Every human-facing item has a **cost estimate** and a **class** (below).
- The system **schedules** attention the way the [heartbeat](heartbeat.md) schedules workers: batched windows, priority ordering, caps.
- Exceeding the budget is a **guard trip, not an overtime request** — excess items queue to the next window, and chronic overflow is a design signal, never a personal one.

## Interrupt classes

Not every escalation deserves an interrupt. A proposed classification, mirroring the guardrails' layered posture:

| Class | Latency contract | Examples | Delivery |
|---|---|---|---|
| **Page** | Minutes — interrupt now | Circuit breaker storm, budget-cap breach, paradox blocking an active contract deadline | Push notification |
| **Window** | Same day — next review window | Spawn approvals, goal-weight changes, unverified-merge flags | Batched queue, priority-ordered |
| **Digest** | Days — periodic summary | Postmortem triage, idle-company reports, eval-drift notes | Daily/weekly digest |
| **Archive** | None — pull only | Guard-refusal metrics, routine completions | Dashboard, consulted when investigating |

Two rules make the classes real. First, **class is assigned by the emitter and audited by recurrence**: anything that pages more than rarely is either misclassified or a design flaw — the same "recurrence is a signal" rule the guardrails use. Second, **nothing may promote itself**: an agent cannot escalate a Digest item to a Page by re-filing it. Promotion is an operator or governance action.

## Queue mechanics

- **One queue, priority-ordered, cost-annotated.** The operator opens one surface and sees the day's decisions ranked, each with the context needed to decide *inline* — the goal IDs, the contract, the diff, the incompatibility statement. An escalation that requires the operator to go digging has failed its contract; assembling the decision context is the *escalating component's* job, exactly as postmortem assembly is the pipeline's job.
- **Review windows, not ambient trickle.** Attention batches like dispatch does — peak-hour filtering already exists for agent work; the same shape applies to human work. Outside windows, only Page-class interrupts get through.
- **Decision SLAs with safe defaults.** Every queued item declares what happens if it ages out: spawn approvals expire (fail-closed), unverified-merge flags block promotion (fail-closed), digest items roll forward. Nothing "fails open because the human was busy."
- **Metrics or it didn't happen.** Queue depth, item age, decisions per window, time-per-decision by class, and — most important — **deferral and reversal rates**. A rising reversal rate is the vigilance-degradation alarm; it means decisions are being made faster than they are being considered.

## Recurrence is a design signal — here too

The guardrails page establishes that a guard firing twenty times is a design flaw being found. The same reading applies upward:

- A **checkpoint that fires constantly** (every spawn needs approval, forever) is a checkpoint that should be narrowed, delegated to a governance-class company, or converted into an eval-gated fast path.
- A **goal that keeps producing paradox escalations** flags the goal for refinement — [goal-aware coordination](goal-aware-coordination.md) already says this; the attention queue is where the pattern becomes measurable.
- A **digest nobody reads** is telemetry, not communication — demote it to Archive and stop pretending it was reviewed.

The steady state to design for: **the operator's attention spends down on judgment — structure, values, paradoxes — and never on ceremony.** Every ceremonial approval that survives contact with this page should have to justify why it is not an eval.

## What this is not

This is not a proposal to remove human checkpoints — the three human decision points (spawn, paradox, extreme weight change) stay exactly where [governance](../architecture/governance.md) put them. It is the opposite claim: those checkpoints only retain their value if the human arriving at them has attention left to spend. Budgeting attention is what keeps the checkpoints honest.

Nor is it delegation of judgment to agents. Agents assemble context, classify, batch, and meter; humans decide. The queue is plumbing, not a proxy.

## See also

- [Execution Guardrails](execution-guardrails.md) — the same containment thesis, applied to agents
- [Goal-Aware Coordination](goal-aware-coordination.md) — paradox escalation, the highest-value attention consumer
- [Postmortems](postmortems.md) — triage discipline that feeds the Digest class
- [Heartbeat](heartbeat.md) — the scheduling pattern this page borrows for humans
