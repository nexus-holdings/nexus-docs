# Wings & Rooms

Nexus Memory uses a two-level taxonomy: **wings** at the top, **rooms** inside each wing. This split keeps content addressable, prevents cross-tenant leakage, and makes search scopable.

## The model

```mermaid
flowchart LR
    MP[MemPalace]

    W1[wing: claude<br/><i>human-driven sessions</i>]
    W2[wing: nexus<br/><i>Nexus platform itself</i>]
    W3[wing: &lt;your-product&gt;<br/><i>per-project scope</i>]

    R1[room: sessions]
    R2[room: memory_files]
    R3[room: config]

    R4[room: cockpit]
    R5[room: documentation]
    R6[room: decisions]

    R7[room: plan_docs]
    R8[room: analysis]
    R9[room: scripts]

    D[drawer<br/><i>verbatim content<br/>+ metadata</i>]

    MP --> W1
    MP --> W2
    MP --> W3

    W1 --> R1
    W1 --> R2
    W1 --> R3

    W2 --> R4
    W2 --> R5
    W2 --> R6

    W3 --> R7
    W3 --> R8
    W3 --> R9

    R1 -.-> D
    R2 -.-> D
    R3 -.-> D

    classDef brass fill:#14171A,stroke:#D4A574,color:#E6E3DC,stroke-width:1.5px
    classDef wing fill:#14171A,stroke:#A8AAB0,color:#E6E3DC,stroke-width:1.2px
    classDef other fill:#0D0F11,stroke:#6F7177,color:#E6E3DC,stroke-width:1px
    class MP,D brass
    class W1,W2,W3 wing
    class R1,R2,R3,R4,R5,R6,R7,R8,R9 other
```

Every memory item — a **drawer** — lives in exactly one `(wing, room)` pair. The taxonomy is intentionally flat at two levels: deeper hierarchy turned out to fragment recall more than it helped retrieval scope.

## Wings

A wing is a **scope of authorship**. Use one wing per:

- Product (e.g., `your-product`)
- Tenant (e.g., `customer-acme`)
- Session source (e.g., `claude` for human-driven sessions, `claude-nexus-agents` for dispatched-agent sessions)

Wings are deliberately coarse. Don't create a new wing per ticket or per file — that goes in rooms.

## Rooms

A room is a **semantic bucket** within a wing. Typical room names:

| Room | What goes in it |
|---|---|
| `sessions` | Full Claude Code session transcripts |
| `memory_files` | Auto-memory `.md` notes |
| `plan_docs` | Planning / design markdown |
| `decisions` | ADRs / decision records |
| `documentation` | User-facing or onboarding docs |
| `scripts` | Code snippets / utility scripts |
| `analysis` | Output of investigations |
| `config` | Configuration files |

Rooms are conventional, not enforced. New rooms are created on first write.

## Why two levels (not one, not three)

- **One level (flat)** loses tenant separation. Hard to query "what does product X know" without metadata filters on every drawer.
- **Three levels** (wing/room/shelf, say) ends up arbitrary — drawers already have rich metadata; another taxonomy level adds friction without information.
- **Two levels** matches how humans actually navigate: "go to the X area, look in the Y bucket." Maps cleanly to faceted search.

## Routing rules

When a drawer is written via `/v1/remember`:

```json
{
  "content": "...",
  "wing": "your-product",
  "room": "plan_docs",
  "source_file": "/path/to/source.md"
}
```

The routing decision is the **writer's** responsibility, not Nexus's. The platform doesn't infer wing/room from content — it stores what the caller specifies.

Convention: agents writing on behalf of a company should use that company's wing. Background ingest paths (e.g., session-end hook) use the wing matching the session source.

## Search semantics

Queries can be scoped at three granularities:

| Scope | Returns |
|---|---|
| All wings | Cross-tenant search. Default. |
| `wing` only | Drawers from that wing across all rooms. |
| `wing` + `room` | Tight scope — single bucket. |

```python
# Whole-MemPalace
GET /v1/retrieve?query=...

# One wing
GET /v1/retrieve?query=...&wing=nexus

# One bucket
GET /v1/retrieve?query=...&wing=nexus&room=decisions
```

## See also

- [Memory Protocol](memory-protocol.md) — when to use which scope
- [Memory Layer](../architecture/memory-layer.md) — where wings/rooms sit in the broader stack
- [Nexus Memory](../components/nexus-memory.md) — implementation details
- [AAAK Encoding](aaak.md) — the compressed format used inside drawer content
