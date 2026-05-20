# Substrate vs. Flywheels

<p class="lede">Nexus is a <strong>substrate</strong>, not a framework. This is a positioning claim, but it's also an architectural one — substrates and frameworks behave differently under sustained use, and the difference is what determines whether the system compounds or plateaus.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-19</span>
  <span>Owner: Platform</span>
</div>

## The claim

Nexus is the part of the system that **persists across products**. Any vertical (a contract-negotiation system, a trading-bot stack, a legal-advisory tool) can be built as a flywheel on top of it. Each flywheel that runs leaves the substrate stronger — more memory, more agents in the catalog, more evals in the registry, more ADRs in the decisions log.

This is not how most agentic systems are built. The default shape is "framework plus app" — a library you import, an application you build on top, and a fresh start every time you want a new product. The substrate-plus-flywheels shape is the alternative, and the rest of this page is the argument for why it wins under sustained use.

## Substrate vs. framework

| Property | Framework | Substrate |
|---|---|---|
| **Lifecycle** | You import, you build, you ship | Runs persistently underneath everything |
| **State** | App owns state; framework is stateless | Substrate owns the canonical state |
| **Learning** | Improves only when the framework is upgraded | Improves continuously as work happens on top |
| **Reuse across products** | Each product re-builds from scratch | Substrate's improvements compound across products |
| **Failure mode** | App crashes, framework is fine | Substrate failure takes everything down |

The framework lives in your dependency graph. The substrate lives in your runtime.

Frameworks are *libraries* — you pull them in, you call their functions, you build on top, you own the app. Substrates are *systems* — they run, they have state, you build *into* them, and the system itself is what improves.

## Pipeline vs. flywheel (recap)

This page assumes you've read [The Flywheel](flywheel.md). The short version:

- A **pipeline** turns input into output. It doesn't learn from running.
- A **flywheel** turns input into output *and also turns itself into a better pipeline*. It compounds.

The flywheel shape requires a persistent thing to compound *into* — somewhere to keep training data, somewhere to register new agents, somewhere to record what was learned. That persistent thing is the substrate. **You can't have a flywheel without a substrate.**

## What "substrate" needs to be

For something to function as a substrate (not just a library that pretends to be one), it has to have:

1. **Canonical entity state** — one place to ask "what exists, what state is it in." Without this, agents disagree about reality and the whole loop breaks.
2. **Durable execution** — a way to make work happen reliably, surviving crashes, picking up interrupted state. Without this, learning is lost on every restart.
3. **Persistent knowledge** — somewhere to put what was learned, with retrieval. Without this, lessons evaporate.
4. **Governance** — a way to decide whether work was good. Without this, "improvement" is unmeasurable.

These four requirements are why Nexus has the [four layers](layers-overview.md) it has. Each layer is one of these requirements, made concrete. You couldn't drop any of them without losing the substrate property.

## Why frameworks don't compound

A framework is great for *building* a thing. It's poor at *being* the thing that learns.

Common failure mode: a team picks an agentic framework, builds a product on top, and then realizes the framework doesn't know anything about their domain after six months of running. Every new feature is built from scratch. Lessons from production failures live in human heads, not in the system. The framework upgrades occasionally but doesn't *learn* from being used.

This is fine for one-shot tools. It's catastrophic for systems that are supposed to keep getting better with use.

Frameworks treat *the application* as the durable thing. Substrates treat *themselves* as the durable thing, and applications (flywheels) as cheaper, swap-in/swap-out artifacts that ride on top.

## What competitors do

Most agentic stacks today are point solutions:

- **Per-vertical agent runtimes** — one company builds an agent for sales, another for legal, another for code review. Each rebuilds the basics: memory, dispatch, governance, evals. No sharing.
- **Framework-shaped products** — LangChain, AutoGen, etc. — useful for prototyping but designed to be imported, not deployed-as-persistent-infra.
- **Closed AI platforms** — proprietary, tightly bundled with one vendor's models. You can build apps but you can't own the substrate.

None of these compound. Each new product starts from zero on the substrate-side, even if it imports a familiar framework.

The substrate-plus-flywheels shape is the alternative — one durable layer that *every* product runs on, accumulating value over time.

## External validation

GitLab's [Act 2 announcement](https://about.gitlab.com/blog/gitlab-act-2/) (early 2026) reframed their product as a *substrate* for agentic devops, with a clear "platform that persists, products that ride on top" thesis. The shape they described is structurally identical to what Nexus has been building toward.

That's validation, not novelty — the substrate-plus-flywheels shape is one that anyone working on long-running agentic systems converges on eventually. The interesting question isn't *whether* to build a substrate, it's *whether you can build one before your competitors do.*

## What this means for new product builders

If you're considering building a new agentic product on top of Nexus:

- You get the four layers for free. You don't rebuild memory, dispatch, governance, eval infra.
- Your improvements (new agents, new evals, new ADRs) are *substrate improvements* — they benefit every future product too.
- You're slower in month one (the substrate's discipline takes time to learn) and faster in month six (because the substrate did most of the heavy lifting).

If you're considering building a new agentic product *without* Nexus (or a Nexus-like substrate):

- You'll move faster initially.
- Around month three you'll realize you're rebuilding what Nexus already has.
- Around month nine you'll have built a worse version of Nexus and named it differently.

That's the trade. The substrate cost is paid upfront. The substrate benefit compounds.

## See also

- [The Flywheel](flywheel.md) — the load-bearing thesis this positioning supports
- [Layers Overview](layers-overview.md) — what makes Nexus a substrate, in concrete terms
- [Platform Layer](platform-layer.md) — the foundational layer that makes it work
