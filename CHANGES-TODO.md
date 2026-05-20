# Nexus Docs — outstanding changes

Tracking list captured 2026-05-19. Each item links to its current status and proposed action.

## 1. Cockpit page is outdated

**Current state:** `components/cockpit.md` is a stub. The mental model + cheat sheet still describe Cockpit as a standalone Next.js dashboard.

**Reality:** Cockpit has been superseded by a plugin rendered inside the **governance company**. Trying to merge that reworking is what kicked off the recent cleanup + refactor work.

**Action:**
- [ ] Check main of any `cockpit` / `governance` repos for the new plugin
- [ ] If not on main, query MemPalace (`/v1/search` with `query="cockpit plugin governance"`)
- [ ] If still nothing, defer until next session with the laptop Claude can be queried
- [ ] Update: `components/cockpit.md`, `getting-started/overview.md`, `getting-started/cheat-sheet.md`, `getting-started/installation.md` (remove the Cockpit-on-3000 entries)

## 2. AAAK encoding may be deprecated

**Current state:** `concepts/aaak.md` was written as a full page assuming AAAK is in active use.

**Possible reality:** We may have moved away from AAAK because it's lossy.

**Action:**
- [ ] Query MemPalace for AAAK status: `/v1/search?query="AAAK lossy"` and `/v1/search?query="AAAK deprecated"`
- [ ] If deprecated: rewrite the page as historical context only, mark deprecated, link to the replacement (if any)
- [ ] If still alive: leave page as-is

## 3. Code snippets need inline comments

**Current state:** Code blocks throughout the new pages are bare — no inline comments explaining what each line is doing.

**Action:**
- [ ] Sweep all 11 Phase 1 pages, add `# explanatory comment` lines to every nontrivial code block
- [ ] Adopt as a convention going forward (Phase 2 onward)

## 4. Session ingestion should be a generic "ingest" page

**Current state:** `concepts/session-ingestion.md` focuses on the Claude Code session-end path specifically.

**Desired:** A more general "Ingestion" page covering multiple sources with examples. **Living page** — gets updated as new ingest paths land.

**Action:**
- [ ] Rename `concepts/session-ingestion.md` → `concepts/ingestion.md` (update mkdocs.yml + cross-links)
- [ ] Restructure with sections per source: sessions, memory files, project docs, code, deliberate diary entries, external (Slack/email/etc.)
- [ ] Each section has: trigger, transform, target wing/room, example payload
- [ ] Mark page as "Living document — updated as new ingest paths land"

## 5. More diagrams

**Current state:** Diagrams on landing, tickets, session ingestion. Most other pages text-only.

**Action:**
- [ ] Add mermaid diagrams to: companies (the holdings → subsidiaries → craft model), heartbeat (the loop), wings-and-rooms (taxonomy tree), aaak (entity-relationship glyph IF page survives review)
- [ ] Architecture pages (flywheel, platform-layer, etc.) need diagrams — they're still stubs
- [ ] Components pages should each have a "where it fits" diagram
- [ ] Adopt convention: every non-trivial concept page gets at least one mermaid diagram

## 6. Design system via Claude Design

**Desired:** A coherent design system — modern, stylish, minimal, functional. "Cool docs" feel. Not gaudy or gauche.

**Action:**
- [ ] Write a Claude Design prompt describing the brand position + aesthetic target + references + what to avoid
- [ ] Prompt lives at: `site/design-prompt.md`
- [ ] User feeds prompt into Claude Design → gets a design system back
- [ ] Translate that system into `docs/stylesheets/extra.css` overrides + component patterns

---

## Order to tackle

1. **(Now)** Write the design prompt — unblocks the design conversation
2. **(Now)** Investigate Cockpit + AAAK status in memory — quick queries
3. **(Phase 1.5)** Code-comment sweep across the 11 Phase 1 pages
4. **(Phase 1.5)** Restructure session-ingestion → ingestion
5. **(Phase 1.5)** Diagrams added to existing pages
6. **(Phase 2)** Continue with components, with comments + diagrams from the start
7. **(Once design system arrives)** Apply CSS + component patterns

Phase 2 (components reference) is paused until at least items 1 + 2 + 4 are resolved — we don't want to write 12 more pages and then redo them.
