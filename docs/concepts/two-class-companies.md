# Two-Class Companies

<p class="lede">Every company in Nexus is either <strong>domain</strong> (owns a product or problem space) or <strong>craft</strong> (a stateless execution engine for a single capability). The split is load-bearing — it's what lets one engineering company serve many product companies instead of each product company growing its own engineering team. <a href="decisions-index.md">ADR-032</a> is the source decision.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-19</span>
  <span>Owner: Platform</span>
</div>

## The split

A traditional software company has product teams *and* an engineering department, with the engineering department serving all the product teams. Nexus mirrors that structure at the company level:

| Class | Owns | Knows | Output |
|---|---|---|---|
| **Domain** | A product, service, or cross-cutting concern | The problem space deeply | Well-formed tickets dispatched to craft companies |
| **Craft** | A single execution capability | Standardized processes, repo conventions, the craft itself | Code, instrumentation, research briefs — depending on craft |
| **Governance** | The substrate itself | Strategy, allocation, conflict resolution | Decisions and budgets — sits above both other classes |

Nexus Holdings is the only governance company. Everything else is either domain or craft.

## Why split at all?

The plan looks redundant at first: "why not just have each product company do its own engineering?" The motivation comes from ADR-032's context section — four concrete problems with the unified-company model:

1. **Duplicated capability** — every company building its own engineering team means N redundant tech-leads, N divergent code-review rubrics, N versions of "should we commit + push or just commit?"
2. **Inconsistent execution** — agents in different companies develop different habits. Quality depends on which company's lead wrote the prompt.
3. **Domain contamination** — Lighthouse (a podcast intelligence product) shouldn't need to understand git branching. Nexus Engineering shouldn't need to understand podcast editorial workflows.
4. **Scaling friction** — adding a new product means bootstrapping a full engineering team. Slow, redundant, error-prone.

The two-class model directly answers each:

1. **One craft company serves many domain companies** — one engineering team, one set of conventions
2. **Standardized process inside craft** — everyone goes through Nexus Engineering's review rubric
3. **Hard interface at the ticket** — domain knows the domain, craft knows the craft, the ticket is the API between them
4. **Linear scale-out** — add a new domain company, plug it into existing craft companies

## The two-name convention

The convention is small but load-bearing — class membership is legible at a glance from the name alone:

| Class | Convention | Example | Why |
|---|---|---|---|
| **Craft** | `Nexus [Capability]` | `Nexus Engineering`, `Nexus Observability` | Advertises what it does — anyone can tell what to hire it for |
| **Domain** | Evocative / branded name | `Lighthouse`, `Ledgerly` | Identity *is* the context — the name communicates *what kind of intelligence* the company holds |
| **Governance** | Structural | `Nexus Holdings` | The one holding company that owns all others |

Reading the org chart should never require checking a registry — the name carries the class.

## The interface: well-formed tickets

<img src="../../assets/diagrams/two-class-interface.svg" alt="Two-class company interface — Domain company on the left, Craft company on the right, with a brass Well-formed Ticket node (the schema-enforced interface) between them. Domain decomposes into ticket, dispatched to craft, craft executes producing Output. A dashed flow-back arrow returns from Output to Domain.">


The **ticket is the API**. Everything the craft company needs to do the work must be in the ticket — repo path, branch base, acceptance criteria, codebase context, priority. The [Craft Dispatch plugin](../components/plugins/craft-dispatch.md) enforces this with a schema; tickets missing required fields are rejected at dispatch time.

The implication: domain companies are responsible for *ticket quality*. Garbage in, garbage out. A vague "make it better" ticket is the domain company's bug, not the craft company's.

## What craft companies do *not* do

Craft companies are stateless execution engines. They specifically don't:

- **Hold domain context** — they need every relevant fact in the ticket
- **Decide priorities** — that's the domain company's call, expressed via the `priority` field
- **Negotiate scope** — they accept or reject the spec; they don't propose changes (they escalate)
- **Maintain product roadmaps** — a craft company has one craft, not a product to ship

The "stateless" qualifier is meaningful: a craft company should be able to handle ticket N+1 from a *different* domain company immediately after ticket N from the previous domain company, with no carry-over context required (per-project knowledge bases keep client context segregated, but those are scoped *under* the craft company, not muddled into it).

## What domain companies do *not* do

Inversely, domain companies don't:

- **Write code, conduct research, produce content directly** — agents are product / domain roles (Product Lead, Domain Expert, Analyst), not engineers
- **Maintain craft standards** — they accept what the craft company produces (or escalate via reject)
- **Decompose into implementation details** — they decompose into *engineering tickets* with acceptance criteria; the *implementation* lives in the craft company's hands

A domain agent dispatched to engineering work would be at a disadvantage — it doesn't have the per-craft conventions, the standardized review rubric, or the cross-project knowledge. The split forces work to flow to the right specialist.

## The three output types from domain companies

Per the canonical taxonomy, domain companies produce three kinds of work order:

| Output type | Destination | When |
|---|---|---|
| **Implementation spec** | Nexus Engineering as build ticket | Requirements are clear; ready to implement |
| **Research brief** | Internal | C-suite lacks information to make a decision; resolved before any ticket is created |
| **Health mandate** | Nexus Observability or Nexus Engineering | Non-negotiable foundations — observability gaps, missing tests, infra deficiencies |

Health mandates are the substrate's lever for production-readiness: the domain company can't just ignore "we have no metrics" — it owns the production-readiness of its products, so it has to file the mandate.

## Nexus Engineering — two modes only

The engineering craft company operates in *exactly* two modes, and the mode is explicit in every ticket:

| Mode | Trigger | Output |
|---|---|---|
| **Build** | Spec ticket with acceptance criteria | Feature branch: committed, tested, pushed |
| **Review** | Code or PR reference | Structured findings: pass / fail, issues, recommendations |

A domain company creates one or the other. Never a hybrid — no "build it and also review the surrounding code while you're there" tickets. The single-mode constraint keeps the work scoped and the review rubric applicable.

## Nexus Observability — cross-cutting mandate

Observability is *also* a craft company, but its mandate is different from engineering's. Engineering serves the domain company that filed the ticket; observability serves *the platform*, working across all projects with a coverage mandate.

The mandate:
- Every shipped project has proper instrumentation
- Logging standards are defined and audited across repos
- Alerting + dashboards exist for new projects
- Coverage reports surface what's instrumented vs. what's dark

Any domain company can file observability tickets; so can governance.

## Current org chart

The substrate today (per `docs/company-taxonomy.md`):

| Company | Class | Goal |
|---|---|---|
| **Nexus Holdings** | Governance | Strategy, resource allocation, approvals |
| **Nexus Engineering** | Craft | Build + review code across all projects |
| **Nexus Observability** | Craft | Instrumentation + monitoring + alerting cross-project |
| **Nexus Platform** | Domain | Nexus's own platform products (cockpit, plugins, infra) |
| **self-improvement** | Domain | Evaluate and optimize all companies via metrics + evals |
| **Lighthouse** | Domain | Podcast intelligence + research reports |
| **Ledgerly** | Domain | AI invoice management (not yet staffed) |

Adding a new product means adding a new domain company. The craft companies stay constant.

## When the model strains

Three cases where the two-class split is uncomfortable, with the substrate's current answers:

| Situation | Strain | Resolution |
|---|---|---|
| Domain company wants to debug its own ticket post-completion | Domain wants to look at code; that's craft territory | Domain reads the code (read-only) but doesn't edit; if changes needed → file a new ticket |
| Craft company spots an issue beyond the ticket scope | Should it fix? Escalate? Ignore? | Escalates via review verdict; original domain company decides scope |
| A new capability doesn't fit cleanly into engineering or observability | Neither craft company is the right home | Either extend an existing craft (rare) or create a new craft (e.g. `Nexus QA`, `Nexus Editorial`) |

The pattern: the answer is almost always "keep the split clean and escalate at the seam," not "merge the responsibilities."

## See also

- [Decisions Index](decisions-index.md) — ADR-032 (the source decision)
- [Companies](companies.md) — the entity primitive itself
- [Tickets](tickets.md) — the API between classes
- [Craft Dispatch plugin](../components/plugins/craft-dispatch.md) — the implementation of the cross-class dispatch
- [Contracts plugin](../components/plugins/contracts.md) — the formal-agreement variant (ADR-043 builds on this model)
- [Agent Catalog](../components/agent-catalog.md) — agents are scoped per-company, but the catalog itself is class-aware
