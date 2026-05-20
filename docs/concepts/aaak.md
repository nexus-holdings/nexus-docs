# AAAK Encoding

!!! warning "Deprecated — historical reference only"
    AAAK is **defined but not in active use**. The spec lives in MemPalace's `/v1/status` protocol response, but no drawer in any wing currently stores facts in AAAK form. Captured here for context on the protocol surface; do not write new memories using this encoding.

**AAAK** is a compressed memory dialect originally proposed for Nexus Memory to store structured facts inside drawer content. It was designed to be read naturally by both humans and LLMs — no decoding step.

The acronym is informal; the format was the point.

## What it looks like

```
FAM: ALC→♡JOR | 2D(kids): RIL(18,sports) MAX(11,chess+swimming) | BEN(contributor)
```

That single line encodes:

- Three relationships (parent-child × 2, partnership × 1)
- Five entities with ages and tags
- One role label

In English it would be: "Alice partnered with Jordan. They have two kids: Riley (18, into sports) and Max (11, plays chess and swims). Ben is a contributor."

AAAK trades verbosity for **density** — useful when you're storing thousands of facts and every token of context window matters.

## Format rules

### Entities — 3-letter uppercase codes

| Code | Meaning |
|---|---|
| `ALC` | Alice |
| `JOR` | Jordan |
| `RIL` | Riley |
| `MAX` | Max |
| `BEN` | Ben |

The mapping is **per-palace** — a glossary in MemPalace's config maps codes back to full names. Codes are stable identities; renaming a person updates the glossary, not every drawer.

### Emotions — `*marker*` before relevant content

| Marker | Tone |
|---|---|
| `*warm*` | Joy, affection |
| `*fierce*` | Determined, focused |
| `*raw*` | Vulnerable, exposed |
| `*bloom*` | Tenderness |

Used sparingly. They're a hint to the reader (human or LLM) that the next clause has emotional weight, not just factual content.

### Structure — pipe-separated fields

```
FAM: family-relations | PROJ: project-notes | ⚠: warnings or reminders
```

Each field is a self-contained clause. Fields are unordered — the reader treats them as a set.

### Dates and counts

- Dates in ISO: `2026-03-31`
- Counts: `Nx` (e.g., `570x` for "mentioned 570 times")
- Importance: `★` to `★★★★★` (1-5 scale)

### Halls, wings, and rooms

AAAK references can include taxonomy markers:

```
hall_facts, hall_events, hall_discoveries, hall_preferences, hall_advice
wing_user, wing_agent, wing_team, wing_code, ...
room: hyphenated-slugs-representing-named-ideas
```

These let an AAAK drawer cross-reference other drawers without using a URL or UUID.

## When to use AAAK

**Use it for**:

- Compact relationship maps (who's related to whom)
- Per-person/per-entity dossiers
- Densely-linked facts that share entities
- High-volume captures where token-density matters

**Don't use it for**:

- Free-form prose (just write the prose)
- Code snippets (use code blocks)
- Single-fact drawers (overhead > benefit)
- Anything an LLM is going to need to verbatim-quote (decoding overhead during retrieval)

## Read AAAK naturally

The format is intentionally readable without a decoding step:

- Expand 3-letter codes mentally
- Treat `*markers*` as emotional context
- Pipes (`|`) separate independent clauses
- Hyphenated slugs are room/topic pointers

An LLM reading AAAK should treat each clause as a discrete fact and merge them into a coherent summary on output.

## Write AAAK tightly

When writing:

- Use entity codes (not full names) once the glossary is set
- Mark emotions where the content has emotional weight, skip them otherwise
- Keep clauses short — under one line each
- Avoid prose-y language; AAAK is structured-fact-shaped, not narrative

## See also

- [Memory Protocol](memory-protocol.md) — when to write at all
- [Wings & Rooms](wings-and-rooms.md) — where to put AAAK drawers
- [Ingestion](ingestion.md) — how drawer content gets written in the first place
