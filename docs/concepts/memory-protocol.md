# Memory Protocol

The memory protocol is the **rule set** that turns Nexus Memory from a storage system into actual memory. Without the protocol, you have a vector database. With it, you have an agent that **knows** before it speaks.

## The core rule

> Search before responding about any person, project, or past event. Never guess. Verify.

That sentence is the whole protocol in compressed form. The rest of this page is its operationalization.

## The five disciplines

### 1. On wake-up: load the palace overview

The first thing any agent does in a fresh session is hit `mempalace_status` to pull:

- The list of wings and rooms (taxonomy)
- The AAAK dialect spec (so it can read compressed drawers)
- The current protocol text (the rules below)

This is the agent's "where am I" check. Without it the agent doesn't know what scopes exist or what conventions apply.

### 2. Before any factual claim: search

Before mentioning a name, a relationship, a past decision, a project's status:

```python
results = mempalace_search(
    query="<the topic you're about to discuss>",
    wing=<scope>,            # optional
    max_results=5,
)
```

If results exist, ground the response in them. If they don't, **say so** — "I don't have this in memory" beats confabulating.

### 3. When uncertain: query, don't guess

If you're about to say "I think X is Y" — stop and check. The cost of a 50 ms `/v1/search` round-trip is trivial. The cost of a confident wrong answer is high.

### 4. After each session: write a diary entry

At the end of a session, capture:

- What happened (objective record)
- What was learned (insights)
- What matters going forward (priority signals)

Goes into the `sessions` room of the relevant wing. The session-ingest pipeline does this automatically for transcripts, but **deliberate** diary entries — opinions, decisions, surprises — should be written separately so they're tagged as such.

### 5. When facts change: invalidate, then add

Facts go stale. The protocol distinguishes correction from contradiction:

- **Correction** — "I was wrong about X." Mark the old drawer invalid; write a new one with the corrected fact + a pointer to what it supersedes.
- **Contradiction** — "X used to be true, now it's not." Same operation: invalidate old, add new.

Don't silently overwrite. The old fact has provenance (when, why, who) that matters.

## Why this matters

LLMs hallucinate. They will, with high confidence, tell you things that aren't true — especially about long-tail entities (specific people, specific projects, specific past decisions). Three things help:

1. **Storage** — having facts on disk
2. **Retrieval** — having a fast, accurate way to pull them at query time
3. **Discipline** — *actually using* retrieval, every time, before answering

The platform gives you (1) and (2). The protocol is how you commit to (3).

A useful slogan: **storage is not memory. Storage + this protocol = memory.**

## Failure mode: protocol drift

Agents will, over many turns, gradually drift from the protocol. They'll start guessing instead of searching. They'll skip the wake-up call. They'll write diary entries less often.

Two anti-drift mechanisms:

1. **Protocol-in-context** — every dispatched session prepends the protocol text to the system prompt
2. **Eval gaps** — the eval registry flags sessions that produced confident factual claims without any search call in the transcript

## Anti-pattern: scope leakage

Don't search the wrong wing. If an agent dispatched on behalf of `company-A` searches without a `wing` filter, it gets results from `company-B` too. The agent then "knows" things it shouldn't.

Always scope to the wing matching the dispatching company unless you have an explicit reason to cross wings.

## See also

- [Wings & Rooms](wings-and-rooms.md) — the scope mechanism
- [AAAK Encoding](aaak.md) — how compressed facts are stored
- [Ingestion](ingestion.md) — the write side, source by source
- [Nexus Memory](../components/nexus-memory.md) — what runs underneath
