# Memory Layer

<p class="lede">The memory layer is where Nexus keeps everything that needs to survive a single agent session: transcripts, decisions, postmortems, code snapshots, the project notes a session decided were worth saving. It's the substrate's institutional knowledge.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-19</span>
  <span>Owner: Platform</span>
</div>

## What it does

Three responsibilities, no more:

1. **Persist** content emitted during agent work — session transcripts, memory notes, code, decisions
2. **Scope** that content into a navigable namespace (wings → rooms → drawers) so future agents and humans can find it
3. **Retrieve** relevant drawers given a query — both keyword (filename/title) and semantic (embedding similarity)

That's it. The memory layer doesn't *interpret* what's stored, doesn't *act* on it, doesn't *reason*. It's a content store with a smart index.

## The two-store model

<img src="../../assets/diagrams/memory-layer-flow.svg" alt="Memory layer two-store model — MemPalace (write side, JSON on disk) on the top left, Context-1 (read side, embedded vectors) on the top right, connected by a brass promoter arrow. An 'Agent / hook' actor at bottom-left writes up to MemPalace via /v1/remember; an 'Agent' actor at bottom-right reads from Context-1 via /v1/retrieve, with a direct-lookup route also reaching MemPalace.">


The memory layer separates **writes** from **reads** by deliberate design:

- **MemPalace** is the authoritative write-side. Every drawer lands here first as content + metadata. Content-addressed (sha256 over canonicalized body), so duplicate ingests are no-ops. No embedding work happens on write.
- **Context-1** is the read-side. A background process (the promoter) takes drawers from MemPalace, computes embeddings, and writes them to a vector store (ChromaDB) keyed by drawer ID.

Why two stores: ingest is bursty and shouldn't block on embedding (which is GPU work). Read is latency-sensitive and benefits from a pre-computed vector index. Splitting them lets each side scale on its own constraints.

## The taxonomy

Memory is organized as a three-level namespace:

| Level | Purpose | Example |
|---|---|---|
| **Wing** | Top-level partition by *agent* or *project* | `claude`, `nexus`, `aurelius` |
| **Room** | Sub-partition by *content kind* within a wing | `sessions`, `decisions`, `plan_docs` |
| **Drawer** | A single content item (one file's worth) | A single session transcript, one ADR |

A drawer is the atomic unit. Every drawer has:

- A **content** body (text, often a JSON-serialized session transcript)
- A **metadata** record (wing, room, source path, content hash, ingest timestamp)
- A **stable ID** derived from content hash + scope

See the [Wings & Rooms](../concepts/wings-and-rooms.md) concept page for the full taxonomy and the live `mempalace.yaml` for the authoritative wing/room list.

## The promoter

The promoter is the bridge from MemPalace to Context-1:

```python
# Runs on a systemd timer (mempalace-promoter.timer, 5-min cadence).
# Pulls any MemPalace drawers not yet in Context-1, computes embeddings,
# and writes them to ChromaDB. Idempotent: re-running over a synced state
# is a no-op.
def promote_one_wing(wing: str) -> int:
    mp_drawers = mempalace.list_drawers(wing=wing)              # source of truth
    c1_ids     = context1.list_ids(wing=wing)                   # already-mirrored
    pending    = [d for d in mp_drawers if d.id not in c1_ids]  # delta only

    for drawer in pending:
        vector = bge_m3.embed(drawer.content)                   # GPU work
        context1.upsert(drawer.id, vector, drawer.metadata)

    return len(pending)
```

Three properties matter:

- **Incremental** — only newly-arrived drawers are embedded. A full backfill is a one-time cost.
- **GPU-accelerated** — the embedder (bge-m3) runs on local GPU. Single-drawer embed takes ~30ms; CPU fallback is ~1s.
- **Lockfile-guarded** — a PID lockfile at `/tmp/promoter-{wing}.lock` prevents concurrent runs (the Stop hook can fire faster than a slow promoter completes).

## Embedding

Nexus uses **bge-m3** as the embedding model — a multilingual dense embedder with 1024-dim output. Two characteristics drove the choice:

- **High recall on technical text** — outperforms sentence-transformers/all-MiniLM on session-transcript retrieval benchmarks
- **Local-friendly** — runs on a single consumer GPU at sub-second-per-document speed; no API dependency

The embedder is loaded once per promoter run (model load ~5s, then ~30ms per drawer). Switching models would require re-embedding the entire Context-1 corpus, so the choice is locked-in for the medium term.

## Retrieval

Two query modes:

| Mode | Endpoint | When |
|---|---|---|
| **Direct** | `GET /v1/drawers/{id}` against MemPalace | You already know the drawer ID (from a session log, a prior search) |
| **Semantic** | `POST /v1/retrieve` against Context-1 | You have a query string and want the top-K similar drawers |

Semantic retrieval returns `(drawer_id, similarity_score, metadata)` triples. The agent then fetches full content from MemPalace using the IDs. This keeps the vector store small (no full content, just embeddings + metadata).

## What it does not do

The memory layer **does not**:

- **Run agents** — that's the [execution layer](execution-layer.md). Memory is queried by agents; it doesn't spawn them.
- **Decide what's worth remembering** — that's the calling code's job. The Stop hook ingests every session transcript; the agent decides what to write as a memory note. The memory layer just stores what it's told to.
- **Reason about content** — no LLM lives inside the memory layer. The embedder is a single-pass model, not a chat model. If you want a question answered *from* memory, that's an agent's job that uses memory as input.
- **Track platform state** — ticket statuses, company configs, plugin registry live in the [platform layer](platform-layer.md). Memory may reference them by ID but doesn't own them.

This is what keeps the memory layer fast and predictable. Drift in scope here would couple it to every other layer.

## Implementation

The memory layer is implemented by [Nexus Memory](../components/nexus-memory.md), which bundles:

- **MemPalace** — a JSON-on-disk store with a FastAPI surface on port `8102`
- **Context-1** — a ChromaDB instance with the bge-m3 embedder, exposed via the same FastAPI on `/v1/retrieve`
- **The promoter** — a Python script run by `mempalace-promoter.timer`
- **MCP server** — exposes 25 memory tools (`mempalace_status`, `mempalace_search`, `mempalace_diary_write`, etc.) for Claude Code agents

Common config:

```bash
# Where MemPalace stores drawers on disk.
MEMPALACE_DIR=$HOME/.local/share/mempalace

# ChromaDB endpoint (Context-1's vector store).
CHROMADB_HOST=127.0.0.1
CHROMADB_PORT=8101

# GPU device for the bge-m3 embedder. Set to "cpu" to fall back.
BGE_M3_DEVICE=cuda
```

## See also

- [Wings & Rooms](../concepts/wings-and-rooms.md) — the namespace concept in detail
- [Memory Protocol](../concepts/memory-protocol.md) — the read/write API
- [Ingestion](../concepts/ingestion.md) — how content from every source lands in MemPalace
- [Nexus Memory](../components/nexus-memory.md) — the implementation
- [API — MemPalace](../reference/api-mempalace.md) — endpoint reference
- [Platform Layer](platform-layer.md) — what owns entity state (not memory)
- [Execution Layer](execution-layer.md) — what writes to memory at session end
