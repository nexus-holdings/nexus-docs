# Content audit — Phase 1 pages

Pass completed 2026-05-19 against: paperclip live API, nexus-core scripts, paperclip-plugin-acp source, systemd units, mempalace.yaml, ticket-lifecycle.md, state-machines.md, company-taxonomy.md.

---

## Confirmed accurate ✓

- **Service ports**: PostgreSQL 5432, Ollama 11434, Meridian 3456, Paperclip 3100, Cockpit 3000, ChromaDB 8101, MemPalace API 8102. All verified against systemd units + live API.
- **Paperclip binary path**: `/home/ian/.npm-global/bin/paperclipai` ✓ (per `paperclip.service` unit)
- **Data dir**: `/home/ian/Projects/nexus/paperclip/data` ✓
- **`ANTHROPIC_BASE_URL=http://127.0.0.1:3456`** is the Meridian proxy ✓
- **Ticket states** (`backlog/todo/in_progress/in_review/done/cancelled/blocked`) — all confirmed against `ticket-lifecycle.md` + `state-machines.md`
- **`done = merged to main`** — confirmed canonical
- **Heartbeat 5-min cadence** — confirmed in `nexus-heartbeat.timer` (`OnUnitActiveSec=5min`)
- **Peak-hour filter** real — `is_peak_hour()` in nexus.config, cap reduction + priority filter in heartbeat script
- **Sonnet-eligible labels** (`mechanical`, `parallel-safe`, `scaffold`, `chore`) — confirmed in `paperclip-plugin-acp/src/model-selection.ts:27`
- **`NEXUS_MAX_LIVE_WORKERS`**, **`NEXUS_REQUIRE_MEMORY_INFRA`**, **`NEXUS_IDLE_ESCALATION_DISABLED`** env vars — all confirmed real in `company_heartbeat.py`
- **GPU bge-m3 + lockfile guard + Stop hook** — built by us this session, fully accurate
- **4-wing taxonomy** (claude / claude-nexus-agents / negotiagent / nexus) — confirmed via live MemPalace `/v1/status`
- **Two-class company model** (craft + domain) — confirmed in `company-taxonomy.md`

---

## Errors to fix ✗

### 1. Session-cap default is misleading

- **What I wrote** (cheat-sheet, companies, heartbeat): "Per-company limit on concurrent in-flight sessions. Default 2."
- **Reality**: Two different caps exist:
  - **Per-company cap** = DB-driven via `session_allocations` table; `DEFAULT_CAP = 2` if no row
  - **Global concurrent workers cap** = env var `NEXUS_MAX_LIVE_WORKERS`, **default 1** (production setting per Phase-1 ramp)
- **Fix**: clarify both caps and current values

### 2. Cockpit is archived / pending rewrite

- **What I wrote** (overview, installation, first-ticket, cheat-sheet): described Cockpit as a current Next.js dashboard for live session transcripts + governance UI.
- **Reality**: per live Paperclip API, **company status = `archived`**. Per `project_cockpit_rewrite.md` (memory), full rewrite is pending. New shape is a plugin rendered inside the governance company (per user; not yet in memory).
- **Fix**: mark as deprecated / pending rewrite everywhere; soften the "open Cockpit to watch the agent work" prose

### 3. Reviewer verdicts oversimplified

- **What I wrote** (tickets): "Three verdicts: Approve / Reject / Escalate."
- **Reality**: Code has explicit `approved: boolean` (binary) plus `in_review → todo` (reject) transition. "Escalate" is a real informal option — reviewer can leave ticket in `in_review` and tag a human — but it's not a first-class code construct.
- **Fix**: soften to "approve / reject, with escalation as a third manual path"

### 4. AAAK encoding is aspirational, not in use

- **What I wrote**: full page describing AAAK as the compressed dialect for storing facts.
- **Reality**: AAAK spec lives only in MemPalace's `/v1/status` protocol response. Zero memory drawers actually use AAAK encoding (verified by content search). It's a defined-but-unused protocol.
- **Fix**: add prominent "deprecated / not in active use" notice at the top of `aaak.md`; remove AAAK references from cheat-sheet's "concepts" line

---

## Unverified — leave as-is until human eyes ⚠

- The exact wording of completion-comment format (`[DONE] ...`) I showed in first-ticket — likely accurate but couldn't find it verbatim in source
- "Branch named `ticket/<shortId>`" — referenced in heartbeat output we saw earlier and in status-machine, accurate
- "Acceptance criteria" header convention — agents do look for this but the exact regex isn't documented; soft claim

---

## Recommendation

Fix items 1–4 immediately (small targeted edits, ~15 min). The unverified items in ⚠ are sound enough to leave; revisit if a future error surfaces them.

---

## Applied 2026-05-19

- ✓ `concepts/tickets.md` — reviewer verdicts softened (binary in code + escalate by convention)
- ✓ `concepts/aaak.md` — added deprecated/historical callout at top
- ✓ `getting-started/cheat-sheet.md` — session-cap nuance, AAAK marked deprecated, Cockpit marked pending-rewrite
- ✓ `concepts/heartbeat.md` — pseudocode shows global + per-company cap; table entry rewritten
- ✓ `concepts/companies.md` — session-cap entry rewritten with both layers
- ✓ `getting-started/overview.md` — Cockpit table cell annotated
- ✓ `getting-started/installation.md` — Cockpit step gated behind warning callout; service table cell annotated
- ✓ `getting-started/first-ticket.md` — "watch the agent work" reframed around JSONL tail; Cockpit reference moved to "legacy"

---

## Phase A new pages — written 2026-05-19

| Page | Status | Source material | Notes |
|---|---|---|---|
| `architecture/flywheel.md` | ✓ written | `nexus-integration/architecture/04-data-flywheel-architecture.md` + landing-page positioning | Generalized substrate+flywheels thesis; Negotiagent used as worked example. Structural SVG diagram hand-crafted with the structured-diagram skill. Forward-references `substrate-vs-flywheels.md` (coming in Chunk 3). |
| `architecture/platform-layer.md` | ✓ written | Phase 1 ticket/company pages + DFA references | Frames the layer as canonical entity-state record; Paperclip as implementation. |
| `architecture/execution-layer.md` | ✓ written | DFAs 1, 4, 5, 6 in `docs/state-machines.md` + `nexus-core/README.md` + `concepts/heartbeat.md` | Maps the four state machines; describes dispatch loop with env-var clarification carried over from the Phase 1.5 audit. |
| `architecture/memory-layer.md` | ✓ written | `mempalace.yaml` + Phase 1 wings-and-rooms + this-session promoter/GPU work | Two-store model (MemPalace write-side + Context-1 read-side); promoter pseudocode; bge-m3 embedder rationale. Forward-references `nexus-memory` component + `api-mempalace` reference. |
| `architecture/governance.md` | ✓ written | DFA 20 (postmortems) + ADR-032/-043 references | Four primitives: evals, postmortems, ADRs, contracts. ADR sample shown. Forward-references `decisions-index.md` + `postmortems.md` (Phase C). |
| `architecture/layers-overview.md` | ✓ written (NEW gap page) | The four existing layer pages + landing-page mental model | One-screen summary of all four layers + dependency rule (lower layers don't read higher). Serves as the Architecture tab entry point. |
| `architecture/substrate-vs-flywheels.md` | ✓ written (NEW gap page) | Memory notes (GitLab Act-2 reframing) + this-session positioning discussions | The "why this shape" page. Substrate vs framework comparison, why frameworks don't compound, what competitors do, GitLab Act-2 external validation. |

---

## Diagram upgrades (hand-crafted SVG via structured-diagram skill)

| Diagram | Source page | Replaces | Notes |
|---|---|---|---|
| `flywheel-structural.svg` (inline) | `architecture/flywheel.md` | Mermaid mess (auto-layout failures) | L-shape; chained agent generations; 4-subsystem detail |
| `platform-layer-shape.svg` | `architecture/platform-layer.md` | Mermaid auto-layout | Memory above, Platform 3x2 grid + brass border in middle, Exec + Gov below |
| `layers-overview.svg` | `architecture/layers-overview.md` | Mermaid criss-cross | Governance outer container with Exec + Platform inside; Memory above; 3 distinct arrows; no crossings |
| `overview-mental-model.svg` | `getting-started/overview.md` | Mermaid with deprecated Cockpit + wrong styling | Foundation + towers: brass substrate at bottom with 4 layers as inner boxes; 3 flywheel columns rising out with bidirectional arrows |
| `tickets-flow.svg` | `concepts/tickets.md` | Mermaid with overlapping `escalate` self-loop + `reject` backward arrow | Hand-routed state diagram; happy path along top; blocked off-path; reject + unblock route through margins; escalate dropped (prose says it's manual-convention only) |

Build status: clean except for a forward link to `architecture/substrate-vs-flywheels.md` (Chunk 3 of Phase A — expected).
