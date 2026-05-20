# Create a Company

<p class="lede">This guide walks you through provisioning a new company end-to-end: charter, repo, Paperclip record, agent roster, and a smoke-test ticket that proves the heartbeat can dispatch against it.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-20</span>
  <span>Owner: Platform</span>
</div>

!!! info "Prerequisites"
    - Paperclip is running on `127.0.0.1:3100` (verify with `curl -s http://127.0.0.1:3100/api/health`)
    - PostgreSQL is up — `nexus_metrics` reachable
    - `gh` CLI authenticated against the `nexus-holdings` GitHub org
    - The heartbeat timer is enabled — `systemctl --user is-active nexus-heartbeat.timer`
    - `$NEXUS_ROOT` points at `~/Projects/nexus/`

## Two classes of company

[ADR-032](../concepts/decisions-index.md) splits companies into two classes. Pick one before you start.

| Class | What it is | Examples |
|---|---|---|
| **Domain** | Owns a problem space; decomposes goals into tickets; delegates execution | Aurelius, Ledgerly, Lighthouse |
| **Craft** | Stateless execution engine; receives well-formed tickets, returns outputs | Nexus Engineering, Nexus Observability |

The walked example below provisions a **domain** company (template `product`). Differences for craft are noted at the end.

## 1. Pick name + mission

```bash
# what this is — pick a slug-safe name and a one-line mission
export COMPANY_NAME="Acme Demo"
export COMPANY_MISSION="A throwaway domain company used to validate the provisioning flow"
export COMPANY_TYPE=product   # product | internal | strategic

# verification: confirm the slugify outcome
python3 -c "import re; n='$COMPANY_NAME'.lower(); print(re.sub(r'^-|-$','', re.sub(r'[^a-z0-9]+','-', n)))"
# → acme-demo
```

## 2. Dry-run the provisioning script

`provision_company.py` is the canonical entry point. It is idempotent — running twice will not create duplicates.

```bash
cd "$NEXUS_ROOT/nexus-core"

# what this is — dry-run prints every action without touching anything
uv run python scripts/provision_company.py \
    --name "$COMPANY_NAME" \
    --mission "$COMPANY_MISSION" \
    --type "$COMPANY_TYPE" \
    --dry-run

# verification: output should contain "*** DRY RUN ***" and seven numbered step lines
```

If anything in the dry-run looks wrong (slug, mission, template type, MCP servers, staff roles), fix it before running for real.

## 3. Run it for real

```bash
# what this is — actually create the repo, Paperclip company, CLAUDE.md, and staff agents
uv run python scripts/provision_company.py \
    --name "$COMPANY_NAME" \
    --mission "$COMPANY_MISSION" \
    --type "$COMPANY_TYPE"

# verification: final block prints "Company provisioned: <name>", a repo URL, and a Paperclip URL
```

The seven steps the script runs:

1. Create the GitHub repo from `nexus-holdings/company-template` and clone to `~/Projects/nexus-holdings/<slug>/`
2. Customize wiki + write `wiki/decisions/001-company-charter.md`
3. Populate `CLAUDE.md` from the template-type config
4. Register the company in Paperclip (`POST /api/companies`)
5. Auto-staff persistent agents (Company Lead, Tech Lead — depending on type)
6. Write `.mcp.json` with `github`, `filesystem`, and optionally `paperclip` MCP servers
7. Validate MCP server commands resolve in `PATH`

## 4. Verify the company appears in Paperclip

```bash
# what this is — pull the live list and find your row
curl -s http://127.0.0.1:3100/api/companies \
  | python3 -m json.tool \
  | grep -B1 -A2 "$COMPANY_NAME"

# verification: you should see a JSON object with "name": "Acme Demo" and an "id" UUID
export COMPANY_ID=<paste-uuid-from-above>
```

## 5. Confirm the repo + CLAUDE.md landed

```bash
# what this is — local clone path is ~/Projects/nexus-holdings/<slug>/
ls ~/Projects/nexus-holdings/acme-demo/
head -20 ~/Projects/nexus-holdings/acme-demo/CLAUDE.md
cat ~/Projects/nexus-holdings/acme-demo/wiki/decisions/001-company-charter.md | head -30

# verification: CLAUDE.md exists, contains "## Team Structure", and the charter ADR references the mission
```

## 6. Wire up the agent roster

The provisioning script auto-staffs Company Lead + Tech Lead. Additional execution agents come from the [Agent Catalog](../components/agent-catalog.md) — they are not stored in the company; the company opts in.

To check what's currently staffed:

```bash
curl -s "http://127.0.0.1:3100/api/companies/$COMPANY_ID/agents" \
  | python3 -m json.tool
```

To add catalog agents (e.g. `backend-engineer`), follow [Create an Agent](create-an-agent.md) and reference the agent ID in your roster PR.

## 7. File a smoke-test ticket

```bash
# what this is — minimal feature ticket the heartbeat will pick up on the next tick
curl -X POST "http://127.0.0.1:3100/api/companies/$COMPANY_ID/issues" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Smoke test: create README.md",
    "description": "Add a one-paragraph README.md in the repo root.\n\n## Acceptance criteria\n- README.md exists at repo root\n- Contains the company mission verbatim",
    "status": "todo",
    "priority": "low",
    "labels": ["mechanical"]
  }' | python3 -m json.tool

# verification: response contains "identifier" (e.g. "ACME-1") and "status": "todo"
```

## 8. Watch the heartbeat dispatch it

```bash
# what this is — tail the heartbeat journal to see the dispatch decision
journalctl --user -u nexus-heartbeat -f

# verification: within 5 minutes you should see "<slug>: dispatching ACME-1 (priority=low, label=mechanical)"
```

If you don't want to wait, fire the heartbeat manually:

```bash
systemctl --user start nexus-heartbeat.service
```

The ticket transitions `todo → in_progress`. Follow [First Ticket](../getting-started/first-ticket.md) from this point if you want to see it through to `done`.

## Craft companies differ in three ways

If you pick `--type strategic` (the closest current proxy for "craft" in the provisioning script) instead of `product`:

| | `product` (domain) | `strategic` (closest to craft) |
|---|---|---|
| Session budget | 4 | 2 |
| Staff roles | Company Lead + Tech Lead | Company Lead + Tech Lead (persistent only) |
| MCP servers | github + filesystem + paperclip | github + filesystem + paperclip |
| Issue origin | Files its own work | Receives dispatched work from domain companies via [craft-dispatch](../components/plugins/craft-dispatch.md) |

The provisioning script does not yet have a first-class `--type craft` flag — track its addition in the catalog. For now, use `strategic` and rely on the craft-dispatch plugin to route work in.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `gh repo create` fails with `template not found` | Not authenticated against the `nexus-holdings` org | `gh auth status`; re-auth if needed |
| Paperclip returns `500` from `POST /api/companies` | Paperclip not actually up, or DB unreachable | `curl :3100/api/health`; `journalctl -u paperclip -n 50` |
| Provisioning succeeds but heartbeat never dispatches | Heartbeat timer disabled, or the ticket is still in `backlog` | `systemctl --user list-timers nexus-heartbeat.timer`; `PATCH /api/issues/<id>` to `{"status":"todo"}` |
| Charter ADR missing after re-run | Script's idempotency check sees `001-company-charter.md` and skips wiki customization | Delete the existing file or accept the existing one |
| Auto-staffing reports "Could not staff … (non-fatal)" | Paperclip's agent endpoint rejected the body shape | Re-run; if it persists, file the agent manually via the Paperclip UI |

## See also

- [Two-Class Companies](../concepts/two-class-companies.md) — the domain/craft distinction in depth
- [Companies](../concepts/companies.md) — the concept page
- [Paperclip API — Companies](../reference/api-paperclip.md#companies) — endpoint reference
- [Create an Agent](create-an-agent.md) — staffing a new role on top of the roster
- [First Ticket](../getting-started/first-ticket.md) — end-to-end ticket walkthrough
- [Decisions Index](../concepts/decisions-index.md) — ADR-032 (two-class) and related governance ADRs
