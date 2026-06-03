# The Flywheel

<p class="lede">Nexus is a <strong>substrate</strong> on which any number of <strong>flywheels</strong> run. Each flywheel is a self-improving agentic vertical; the substrate is the part that gets stronger every time one of them turns. This is the load-bearing thesis everything else in Nexus hangs off.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-19</span>
  <span>Owner: Platform</span>
</div>

## The shape

```mermaid
flowchart LR
    SUB[Nexus — substrate<br/><i>platform · execution · memory · governance</i>]
    F1[Flywheel A<br/><i>Aurelius</i>]
    F2[Flywheel B<br/><i>Lighthouse</i>]
    F3[Flywheel C<br/><i>Ledgerly</i>]

    SUB --> F1
    SUB --> F2
    SUB --> F3
    F1 -.outcomes.-> SUB
    F2 -.outcomes.-> SUB
    F3 -.outcomes.-> SUB

    classDef sub fill:#14171A,stroke:#D4A574,color:#E6E3DC,stroke-width:2px
    classDef fly fill:#0D0F11,stroke:#6F7177,color:#E6E3DC,stroke-width:1px
    class SUB sub
    class F1,F2,F3 fly
```

The box on the left runs once. Each box on the right is a separate vertical product running on top, exchanging outcomes back so the substrate (and the next flywheel that lands on it) gets better.

## What "substrate" means here

A substrate is the part of the system that **persists across flywheels**. In Nexus, the substrate has four layers:

| Layer | Provides | Concrete |
|---|---|---|
| **Platform** | Identity, state, scoping — the canonical "what exists" | [Paperclip](../components/paperclip.md) — companies, tickets, agents, plugins |
| **Execution** | Turning intent into action — agents run here | [Nexus Core](../components/nexus-core.md) — heartbeat, dispatch, sessions |
| **Memory** | Knowledge that survives a session | [Nexus Memory](../components/nexus-memory.md) — wings, rooms, drawers, embedded search |
| **Governance** | The rules — evals, postmortems, ADRs, contracts | Plugin model + decision log + postmortem pipeline |

A flywheel that runs on Nexus doesn't bring its own ticketing, its own dispatcher, its own memory store. It uses the substrate's. That's what makes flywheels cheap to add.

## What "flywheel" means here

A flywheel is a vertical that's **closed-loop**. It looks like this:

```mermaid
flowchart LR
    INGEST[Ingest<br/><i>real signal</i>]
    REASON[Reason<br/><i>agents act</i>]
    OUTCOME[Outcome<br/><i>measured</i>]
    LEARN[Learn<br/><i>signal back</i>]

    INGEST --> REASON
    REASON --> OUTCOME
    OUTCOME --> LEARN
    LEARN -.improves.-> REASON
    LEARN -.feeds.-> INGEST

    classDef node fill:#0D0F11,stroke:#D4A574,color:#E6E3DC,stroke-width:1.5px
    class INGEST,REASON,OUTCOME,LEARN node
```

The defining property: **every rotation makes the next rotation better**. Pipelines plateau. Flywheels compound.

A pipeline turns input X into output Y. A flywheel turns input X into output Y *and* also turns the pipeline itself into a better pipeline for the next X. The improvement signal comes from observed outcomes — not from a human re-tuning prompts every week.

## The closed loop

Three things flow back from the flywheel to the substrate after each rotation:

- **Training data** — every session's transcript, agent decisions, and tool calls are captured into [Nexus Memory](../components/nexus-memory.md). Over time this is the corpus that any next-generation agent learns from.
- **Agent improvements** — when GRPO or supervised fine-tuning produces better weights for an agent in flywheel A, those weights are checked into the [Agent Catalog](../components/agent-catalog.md) and become available to flywheel B for free.
- **Eval refinements** — gaps surfaced by postmortems land in the [Eval Registry](../components/eval-registry.md). A failure mode discovered in vertical A becomes a regression test that vertical B inherits.

This is why "the substrate gets stronger" isn't a metaphor — it's a measurable property. Eval coverage, agent skill score, memory drawer count, and ADR depth all go up monotonically across flywheel rotations.

## Why "flywheel" beats "pipeline"

The single-pipeline shape was the standard agentic architecture for two years. It looks like:

```
data → preprocess → LLM → postprocess → output
```

Three problems with it:

1. **No improvement signal** — the pipeline doesn't know whether its output was good unless a human grades it.
2. **No reuse** — a second pipeline for a different vertical shares nothing with the first.
3. **No compounding** — every improvement is a manual prompt-tweak; quality plateaus at "what the current model could do on its first attempt."

The substrate-plus-flywheels shape solves all three:

1. Outcomes are observable (tickets close, contracts settle, customers churn), so the improvement signal exists.
2. Memory, agents, and evals are shared across flywheels — vertical N reuses vertical N-1's substrate improvements.
3. Each flywheel rotation makes the *next* one start from a higher floor.

## Example: a multi-arm flywheel

A flywheel's internals are vertical-specific — what matters here is the *shape*, which generalizes. A flywheel can run several **arms** that all feed one training pool, and a rotation turns that pool into a better generation of agents:

```mermaid
flowchart LR
    A0[Agents<br/>gen N] --> ARM0[Arms emit<br/>training signal]
    ARM0 --> A1[Agents<br/>gen N+1]
    A1 --> ARM1[Arms emit<br/>training signal]
    ARM1 --> A2[Agents<br/>gen N+2]
    A2 --> DOTS[ ... ]

    classDef agents fill:#14171A,stroke:#D4A574,color:#E6E3DC,stroke-width:1.5px
    classDef arms fill:#0D0F11,stroke:#6F7177,color:#E6E3DC,stroke-width:1px
    classDef dots fill:none,stroke:none,color:#6F7177
    class A0,A1,A2 agents
    class ARM0,ARM1 arms
    class DOTS dots
```

One rotation: a training step (for example, GRPO plus a curriculum) trains a new generation of agents on the combined corpus; the fresh weights flow back into the arms; the cycle repeats. The brass-edged agent nodes are the thing that compounds — the arms are the metabolism that turns so the agents can grow.

A flywheel's arms vary by vertical, but a common pattern combines three complementary sources of signal:

- **A simulator arm** — synthetic, two-sided scenarios (abstractly, Company A vs Company B) run against a self-play engine. Because it controls both sides, it produces ground-truth pairs no real-world log can.
- **A reconstruction arm** — recovers the missing side of *one-sided* real data so an existing corpus becomes usable training signal, grounded against the simulator's ground truth.
- **A production arm** — the live deploy target: only the best-performing variants reach it, where they do real work and generate fresh transcripts that feed the next rotation.

Each arm contributes something the others (and the next generation) need: the simulator generates ground truth, reconstruction grounds it in real-world distributions, and production supplies fresh real signal. After one full rotation, the substrate has more memory, better agents in the catalog, sharper evals, and a richer ADR set — and the next flywheel that lands on Nexus inherits all of it.

## Maturity stages

A flywheel doesn't start compounding on day one. Realistic stages:

| Stage | What's happening | Signal |
|---|---|---|
| **0 — Arms exist** | Each arm shipped independently. Loop not closed yet. | Each arm has standalone value. |
| **1 — First rotation** | One end-to-end pass through the loop. | Agents at end of rotation measurably better than at start. |
| **2 — Routine iteration** | Rotations happen on cadence (weekly or per-contract). | Agent quality still climbing; diminishing returns curve is gentle. |
| **3 — Self-bootstrapping** | Synthetic data is most of the training pool; real corpus grounds it but is no longer the bottleneck. | New real data provides marginal improvement, not step-change. |
| **4 — Productized** | The flywheel itself is the product. Ablations are publishable. | Customers buy access to the loop, not just point-in-time output. |

Each stage takes months to years. The decision to build a flywheel rather than a pipeline is a decision to accept a quiet period of investment before the multiplicative phase.

## Failure modes

Flywheels have failure modes pipelines don't. The big four:

- **Reward hacking** — agents discover exploits in the substrate's scoring functions that don't transfer. Mitigation: real-corpus data weighted into training; hold-out evals.
- **Distributional drift** — synthetic data shifts the agent's prior away from real-world distributions. Mitigation: reconstruction or live-corpus anchoring.
- **Confidence cascade** — miscalibrated confidence on one rotation gets baked into the next. Mitigation: periodic re-calibration against held-out ground truth.
- **Feedback amplification** — a pathology in rotation N gets reinforced in N+1, locked in by N+2. Mitigation: anchor corpus that never drifts; postmortem any sudden eval shift.

The substrate gives you the observability to spot these (every transition is logged, every outcome is queryable). It doesn't prevent them — that's a flywheel-design concern.

## See also

- [Substrate vs. flywheels](substrate-vs-flywheels.md) — positioning vs. point-solution stacks
- [Platform Layer](platform-layer.md) — what tracks state
- [Execution Layer](execution-layer.md) — what runs the agents
- [Memory Layer](memory-layer.md) — what persists between rotations
- [Governance](governance.md) — evals, postmortems, ADRs
