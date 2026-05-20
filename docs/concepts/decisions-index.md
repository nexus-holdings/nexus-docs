# Decisions Index

<p class="lede">Every load-bearing decision in Nexus lives as a numbered ADR (Architectural Decision Record) under <code>docs/decisions/</code>. This page is the running index — one line per record, grouped by theme, with status, date, and a one-sentence summary.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-19</span>
  <span>Owner: Platform</span>
</div>

## Why ADRs

An ADR captures three things that get lost over time: **what changed**, **why we changed it**, **what we considered and rejected**. The codebase tells you *what is*; an ADR tells you *what was, and why now is different*. Without them, every six-month-old design decision becomes archaeology.

Two rules make the system useful:

1. **Sequentially numbered, never deleted** — ADR-031 is a postmortem on a specific production failure; even though the bug is long fixed, the record stays so the *reasoning* survives.
2. **Status field is load-bearing** — `Proposed` vs `Accepted` vs `Superseded` tells you whether to trust the record as current. Status is the first thing to update; the record itself never lies about its history.

Postmortems (e.g. ADR-031, ADR-035) are filed *as ADRs* by deliberate convention. The postmortem pipeline (see [Postmortems](postmortems.md)) auto-files them into this same numbered stream.

## Status semantics

| Status | Meaning |
|---|---|
| `Proposed` | Drafted, not yet adopted; may still change before acceptance |
| `Accepted` | In force; the substrate is built / built around this decision |
| `Superseded by ADR-NNN` | No longer authoritative; the linked ADR replaces it |
| `Postmortem` | Records a failure and the resulting learnings; doesn't *decide* but *informs* |

## The index

### Foundations — company model & dispatch

| ADR | Title | Status |
|---|---|---|
| **032** | [Two-Class Company Model](#) | Accepted — domain vs craft companies; the load-bearing distinction at the company level. See [Two-class companies](two-class-companies.md). |
| **033** | [Company Restructuring Plan](#) | Accepted — the multi-phase rollout to bring the substrate onto the two-class model |
| **036** | Domain-Company Pause + Per-Ticket Audit | Accepted — Phase A of the cross-company restructure; the operational pause + audit that preceded the dispatch rewrite |
| **037** | [Cross-Company Ticket Dispatch Mechanism](#) | Accepted — the dispatch protocol. Implemented by [`paperclip-plugin-craft-dispatch`](../components/plugins/craft-dispatch.md). |
| **038** | Engineering Ticket Contract | Referenced (not yet written) — the structured-ticket schema that ADR-037 dispatches conform to |

### Inter-company agreements

| ADR | Title | Status |
|---|---|---|
| **042** | [Ticket-Flag Protocol for Skill Execution](#) | Proposed — the flag protocol that makes ticket-driven skill execution legible to models that drop prose instructions |
| **043** | [Contracts as First-Class Primitive](#) | Accepted — formal cross-company agreements with lifecycle + acceptance criteria. Implemented by [`paperclip-plugin-contracts`](../components/plugins/contracts.md). |
| **044** | Pluggable Negotiation Backend | Forthcoming — referenced from ADR-043 but not yet drafted; the external-counterparty negotiation path |

### Cost, capacity, and quota

| ADR | Title | Status |
|---|---|---|
| **028** | Prompt Cache Optimization Strategy | Accepted — how the substrate uses prompt cache to keep cost predictable |
| **029** | Upstream Session Limit Bug Tracking | Accepted — the tracking record for upstream session-limit quirks; informs the rate-limit DFAs |
| **030** | Direct API Access as Fallback for Token-Intensive Phases | Accepted — when the CLI path is rate-limited, the substrate can fall through to direct API for token-heavy phases |

### Operations & tooling

| ADR | Title | Status |
|---|---|---|
| **034** | Gemma Dispatch Checklist | Accepted — operational checklist for getting Gemma-dispatch tickets running |
| **002** | Session 2 Decisions | Historical — the early-session decision log from April 2026; preserved for archaeology, not authoritative for current design |

### Postmortems

| ADR | Subject | Date |
|---|---|---|
| **031** | Production failure on ticket `87b8c9ab-e4c3-4414-93f7-b244e901f406` | 2026-04 |
| **035** | Domain agents stuck in coding-CLI TUI (chat-adapter misconfiguration) | 2026-04 |

These are filed via the postmortem pipeline (see [Postmortems](postmortems.md)), not hand-written.

## How to write a new ADR

The mechanical version of the process — see [Triage a Postmortem](../guides/triage-a-postmortem.md) for the failure-driven path.

1. **Take the next number.** Look at `ls docs/decisions/` — increment the highest. Even if your number isn't contiguous with the previous one (gaps are fine — they happen when a draft is abandoned).
2. **Start with the template.** Title (`# ADR-NNN: <subject>`), date, status, depends-on, companions, then sections: Context / Decision / Consequences / Alternatives Considered.
3. **Mark dependencies and companions.** "Depends on" = this ADR requires the cited one to be accepted first. "Companions" = related ADRs that together define a coherent design (e.g. ADR-037 + ADR-038 + ADR-042 are the cross-company-dispatch trio).
4. **Status starts at `Proposed`.** It becomes `Accepted` when the build it's gating starts. Updating the status is a real action — don't backdate.
5. **Update this index.** Add a row in the appropriate section.

## What doesn't go in an ADR

Routine bug fixes, refactors, day-to-day code changes — those live in PR descriptions and commit messages. The bar for an ADR is *load-bearing*: a future engineer reading the codebase six months from now would be confused without it.

Specifically:
- **Yes** — choosing TypeScript over Python for plugins (architectural)
- **Yes** — the rule that lower layers don't read from higher layers (invariant)
- **No** — fixing a flaky test (commit message)
- **No** — bumping a dependency version (PR description)

If you're unsure, lean towards writing one. ADR-031 looks routine — it's a postmortem on a single ticket — but its *learnings* changed how the substrate handles silent-termination failures. That's load-bearing.

## See also

- [Postmortems](postmortems.md) — the failure-driven branch of ADR creation
- [Governance](../architecture/governance.md) — the architectural layer that owns the ADR stream
- [Two-class companies](two-class-companies.md) — the concept built on top of ADR-032
- [Contracts plugin](../components/plugins/contracts.md) — implementation of ADR-043
- [Craft Dispatch plugin](../components/plugins/craft-dispatch.md) — implementation of ADR-037
