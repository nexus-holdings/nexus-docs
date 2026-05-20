# MemPalace API

<p class="lede">The Nexus Memory sidecar exposes both a REST surface (FastAPI, port <code>8102</code>) and a pair of MCP servers (Context-1 and MemPalace) for agent use. The REST API is the integration point for plugins and external callers; the MCP servers are the interactive surface that ships inside agent sessions.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-20</span>
  <span>Owner: Platform</span>
</div>

## What it is

A FastAPI server defined in `~/Projects/nexus/nexus-memory/api/server.py` that wraps two backends: Context-1 (ChromaDB-backed agentic retrieval) and MemPalace (the write-side palace store). All endpoints bind to `127.0.0.1:8102` by default. Interactive Swagger UI lives at `http://127.0.0.1:8102/docs`.

| Property | Value |
|---|---|
| **Bind** | `127.0.0.1:8102` (override with `MEMORY_API_PORT`) |
| **Module** | `api.server:app` (FastAPI) |
| **Title** | `Nexus Memory API` |
| **Version** | `0.1.0` |
| **Source** | `~/Projects/nexus/nexus-memory/api/server.py` |

## REST endpoints

| Method | Path | Purpose | Body / query |
|---|---|---|---|
| `GET` | `/health` | Liveness — fast `{"status":"ok"}`. | — |
| `GET` | `/v1/status` | Combined Context-1 + MemPalace status, including collection names and palace counts. | — |
| `GET` | `/v1/wake-up` | Returns L0+L1 wake-up context for the requested wing (shells out to the `mempalace wake-up` binary). | `?wing=<wing>` (optional) |
| `POST` | `/v1/retrieve` | **Primary read path.** Routes through the Context-1 agentic loop, scoped by `wing` (mapped to `mp-<wing>` collection) or explicit `collections`. | `{query, collections?, wing?, room?, max_results}` |
| `POST` | `/v1/search` | Direct MemPalace semantic search — bypass of the agentic loop, retained for debugging and verbatim-match queries. | `{query, wing?, room?, max_results}` |
| `POST` | `/v1/remember` | Store a drawer. Content-hash dedup is always on; optional `(wing, room, source_file)` `replace` for re-ingest paths. | `{content, wing, room, source_file?, source_type?, replace?}` |
| `POST` | `/v1/reindex` | Re-index a directory into a ChromaDB collection. Idempotent upsert with deterministic IDs. | `{directory, collection, source_prefix?, extensions?}` |

### Response shapes

`POST /v1/retrieve` returns:

```json
{
  "results": [
    {
      "id": "…",          // backend-specific id
      "content": "…",     // verbatim drawer/document content
      "source": "…",      // source file or wing tag
      "score": 0.82,      // 0–1, higher = closer
      "backend": "context1",  // "context1" | "mempalace"
      "wing": "nexus",
      "room": "decisions"
    }
  ],
  "count": 1,
  "filters": { "wing": "nexus", "room": null, "collections": ["mp-nexus"] },
  "backends": ["context1"]
}
```

`POST /v1/remember` returns `{stored: bool, detail: str, replaced: int}` (or `{stored: false, reason: "duplicate detected", detail}` when the dedup check fires).

`POST /v1/reindex` returns `{indexed: true, files: <n>, chunks: <n>, collection: <name>}`.

`GET /v1/status` returns `{context1: {healthy, collections} | {healthy: false, error}, mempalace: {healthy, status} | {healthy: false, error}}`.

## MCP tools

Two MCP servers are registered separately. Agents in Paperclip-dispatched sessions reach both surfaces via [`paperclip-plugin-memory`](../components/plugins/memory.md), which proxies a subset; direct registration is also supported.

### Registration

```bash
# Context-1 MCP — 3 tools, retrieval-focused
claude mcp add context1 -- python -m context1.mcp_server

# MemPalace MCP — 25 tools, palace CRUD + knowledge graph
claude mcp add mempalace -- python -m mempalace.mcp_server
```

### Context-1 MCP (3 tools)

Defined in `~/Projects/nexus/nexus-memory/context1/mcp_server.py`.

| Tool | Purpose |
|---|---|
| `retrieve` | Semantic retrieval over ChromaDB collections using the Context-1 agentic search model. Args: `query`, `collections?`, `max_results`. |
| `list_collections` | List all available ChromaDB collections with document counts. |
| `search_collection` | Direct (non-agentic) semantic search on a single named collection. Args: `query`, `collection`, `n_results`. |

### MemPalace MCP (25 tools)

Defined in `~/Projects/nexus/nexus-memory/.venv/lib/python3.12/site-packages/mempalace/mcp_server.py` (PyPI-installed). Grouped by intent:

**Read — drawers**

| Tool | Purpose |
|---|---|
| `mempalace_search` | Semantic search with cosine-distance threshold. Returns verbatim drawer content. |
| `mempalace_get_drawer` | Fetch a single drawer by ID. |
| `mempalace_list_drawers` | Paginated drawer listing, optionally filtered by `wing`/`room`. |
| `mempalace_check_duplicate` | Check if content already exists before filing. |

**Write — drawers**

| Tool | Purpose |
|---|---|
| `mempalace_add_drawer` | File verbatim content into a wing + room. Dedup-aware. |
| `mempalace_update_drawer` | Update an existing drawer's content and/or wing/room. |
| `mempalace_delete_drawer` | Delete a drawer by ID. Irreversible. |

**Read — taxonomy**

| Tool | Purpose |
|---|---|
| `mempalace_status` | Palace overview — total drawers, wing/room counts. |
| `mempalace_list_wings` | All wings with drawer counts. |
| `mempalace_list_rooms` | Rooms within a wing (or all rooms). |
| `mempalace_get_taxonomy` | Full wing → room → drawer-count tree. |
| `mempalace_get_aaak_spec` | The AAAK compressed-memory dialect spec. See [AAAK Encoding](../concepts/aaak.md). |

**Knowledge graph (KG)**

| Tool | Purpose |
|---|---|
| `mempalace_kg_query` | Query an entity's relationships with temporal filtering (`as_of`). |
| `mempalace_kg_add` | Add a triple `(subject, predicate, object, valid_from?)`. |
| `mempalace_kg_invalidate` | Mark a fact as no longer true (`ended`). |
| `mempalace_kg_timeline` | Chronological timeline of facts for an entity (or everything). |
| `mempalace_kg_stats` | KG overview — entities, triples, current vs expired, predicate types. |

**Palace graph (cross-wing tunnels)**

| Tool | Purpose |
|---|---|
| `mempalace_traverse` | Walk the palace graph from a starting room, following tunnel edges. |
| `mempalace_find_tunnels` | Find rooms that bridge two wings. |
| `mempalace_graph_stats` | Palace-graph overview — rooms, tunnels, inter-wing edges. |

**Diary (per-agent journal)**

| Tool | Purpose |
|---|---|
| `mempalace_diary_write` | Write an AAAK-compressed diary entry to the agent's own wing. |
| `mempalace_diary_read` | Read recent diary entries (default last 10). |

**Admin**

| Tool | Purpose |
|---|---|
| `mempalace_hook_settings` | Get or set hook behaviour (`silent_save`, `desktop_toast`). |
| `mempalace_memories_filed_away` | Check if the latest palace checkpoint was saved. |
| `mempalace_reconnect` | Force reconnect after external scripts modified the palace directly (rebuilds the in-memory HNSW). |

## Environment hooks

The API and its backends read these. Full list in [Environment Variables](env-vars.md).

| Var | Default | Read by |
|---|---|---|
| `MEMORY_API_PORT` | `8102` | `api/server.py` |
| `MEMPALACE_BIN` | `<repo>/.venv/bin/mempalace` | `api/server.py` (shell-out for `wake-up`) |
| `MEMPALACE_PATH` | `~/.mempalace/palace` | The `mempalace` package (palace data dir) |
| `CHROMA_HOST` | `localhost` | Context-1 retriever, indexer, promoter |
| `CHROMA_PORT` | `8101` | Context-1 retriever, indexer, promoter |
| `CONTEXT1_BASE_URL` | `http://127.0.0.1:8003/v1` | Context-1 retriever (LLM endpoint for the agentic loop) |
| `CONTEXT1_MODEL` | `~/models/context-1-mxfp4` | Context-1 retriever (model path) |
| `BGE_M3_DEVICE` | `cpu` | Context-1 embedder — set `cuda` for GPU |

## See also

- [Nexus Memory component](../components/nexus-memory.md) — what this API is part of
- [Memory layer](../architecture/memory-layer.md) — the two-store rationale (MemPalace ↔ Context-1)
- [Ingestion concept](../concepts/ingestion.md) — how drawers get into MemPalace in the first place
- [Wings & Rooms](../concepts/wings-and-rooms.md) — the taxonomy these tools operate on
- [Memory plugin](../components/plugins/memory.md) — the 5-tool subset exposed in agent sessions
- [Environment Variables](env-vars.md) — `MEMPALACE_PATH`, `CHROMA_*`, `BGE_M3_DEVICE`, etc.
