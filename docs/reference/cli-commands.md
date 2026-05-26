# CLI Commands

<p class="lede">The terminals into Nexus: installed binaries, Python entry points, and the runbook scripts that ship inside <code>nexus-core/scripts/</code> and <code>nexus-memory/scripts/</code>. Everything below has a verified source path; nothing here is invented.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-20</span>
  <span>Owner: Platform</span>
</div>

## Installed binaries

These live on your `PATH` after installation and stay there.

| Command | Where it lives | What it does |
|---|---|---|
| `paperclipai` | `~/.npm-global/bin/paperclipai` | The Paperclip server. Reads `PAPERCLIPAI_DATA_DIR`, binds `:3100`. Normally run via the `paperclip` systemd unit. |
| `mempalace` | `<nexus-memory>/.venv/bin/mempalace` | The MemPalace CLI (PyPI package). Subcommands: `init`, `mine`, `search`, `compress`, `wake-up`, `split`, `hook`, `instructions`, `repair`, `mcp`, `migrate`, `status`. |
| `nexus-mcp` | `<nexus-mcp>/.venv/bin/nexus-mcp` | Console script registered in `nexus-mcp/pyproject.toml` (`nexus-mcp = "nexus_mcp.server:main"`). The chairman-level MCP server — see [Nexus MCP](../components/nexus-mcp.md). |

## MemPalace CLI

The `mempalace` binary is the most surface-rich CLI in the platform.

```text
mempalace [-h] [--palace PALACE]
          {init,mine,search,compress,wake-up,split,hook,instructions,repair,mcp,migrate,status}
          ...
```

| Subcommand | Purpose |
|---|---|
| `init` | Detect rooms from your folder structure. |
| `mine` | Ingest files into the palace. Two modes: project (`mempalace mine ~/projects/x`) and convos (`--mode convos`). |
| `search` | Find anything, exact-words match. |
| `compress` | Compress drawers with the AAAK dialect (~30× reduction). See [AAAK Encoding](../concepts/aaak.md). |
| `wake-up` | Print L0+L1 wake-up context (~600–900 tokens). The REST sidecar shells out to this. |
| `split` | Split concatenated transcript mega-files into per-session files (run before `mine`). |
| `hook` | Hook-mode worker. Reads JSON from stdin, writes JSON to stdout. Used by Claude Code session hooks. |
| `instructions` | Print skill instructions to stdout. |
| `repair` | Rebuild the palace vector index from stored data (recovers from HNSW corruption). |
| `mcp` | Print the MCP setup command for connecting MemPalace to a client. |
| `migrate` | Migrate a palace across ChromaDB versions (e.g. `3.0.0` → `3.1.0`). |
| `status` | Show what's been filed. |

The `--palace PALACE` flag overrides the default palace location (`~/.mempalace/palace`, or whatever `~/.mempalace/config.json` points at).

## Nexus Core scripts

Python scripts under `~/Projects/nexus/nexus-core/scripts/`. No console-script entry points; run with `uv run` or `python` from the repo root.

| Script | Purpose |
|---|---|
| `company_heartbeat.py` | The heartbeat itself — picks up backlog/todo tickets per company, respects session caps, spawns workers. Run modes: orchestrator (default), `--webhook-fallback`, `--run-ticket TICKET COMPANY`, `--wake`, `--status`. Driven by the `nexus-heartbeat.timer` systemd unit (5-min cadence). |
| `ops_monitor.py` | Stuck-ticket / zombie-process / heartbeat-health detector. Posts escalation issues into Paperclip. Designed to run every ~10 min. |
| `provision_company.py` | Provision a new Nexus Holdings child company end-to-end: GitHub repo + wiki, Paperclip company record, CLAUDE.md, auto-staff Company Lead + Tech Lead. |
| `chairman_cli.py` | Drive Nexus Holdings board meetings via Paperclip issues. Subcommands: `submit`, `run`, `queue`, `review`, `provision`. |
| `nexus_deploy_cli.py` | Deploy agent definitions, skills, and team configs into company repos. Subcommands: `agents <repo>`, `skills <repo>`, `team <repo>`. Source: `agent-catalog/` and `shared-skills/`. |
| `nexus_promote_cli.py` | Promote reusable code from a company repo into `shared-modules` via PR. Enforces the no-cross-company-internals rule. |
| `setup_csuite.py` | Register Nexus Holdings C-suite agents in Paperclip. Reads agent `.md` files and creates Paperclip agent records with `adapterType: "process"`. |
| `ingest_metrics.py` | Tech-Lead metrics ingestion. Accepts ticket completion data via `--json` or flags; validates and writes to `performance_records`. |

```bash
# Common invocations
cd ~/Projects/nexus/nexus-core

uv run python scripts/company_heartbeat.py --status
uv run python scripts/company_heartbeat.py --run-ticket <ticket_id> <company_id>
uv run python scripts/provision_company.py --name "Acme Co" --type product
uv run python scripts/nexus_deploy_cli.py agents /path/to/company-repo --dry-run
```

## Nexus Memory scripts

Python scripts under `~/Projects/nexus/nexus-memory/scripts/`. Run via the project's `.venv`.

| Script | Purpose |
|---|---|
| `promoter.py` | Mirrors MemPalace drawers into Context-1 Chroma collections (`mp-<wing>` per wing). Run via the `mempalace-promoter.timer` (every 15 min) or one-shot with `--incremental`. |
| `hot_snapshot.py` | Online (no-downtime) snapshot using SQLite's backup API. HNSW segments are not captured — restore rebuilds them via `rebuild_hnsw.py`. |
| `cold_snapshot.sh` | Full atomic snapshot to `~/.backups/<timestamp>/`. Briefly stops `mempalace-api.service` to get a consistent capture of palace + HNSW + Context-1 Chroma data. |
| `rebuild_hnsw.py` | Rebuild HNSW indexes after a hot-snapshot restore. Re-paginates every drawer and re-embeds via BGE-M3. ~6 min for 30K drawers on CPU. |
| `migrate_to_bgem3.py` | One-shot migration switching Context-1 collections from MiniLM to BGE-M3. Splits `bench_bgem3` into `mp-<wing>` collections, re-embeds the rest on GPU. |
| `index_<project>.py` | Index a project codebase + design memos into Context-1 (creates/updates the `<project>-code` collection). One script per indexed project — substitute your own project name (e.g. `index_aurelius.py`); the live script in this repo is `index_negotiagent.py`. |
| `cleanup_session_duplicates.py` | One-shot cleanup of session-snapshot duplicate drawers (red-team finding dm-003). |
| `bgem3_bench` | BGE-M3 benchmark harness. |
| `vastai_quantize_context1.py` / `vastai_quantize_context1_llmc.py` | Offline quantisation jobs for the Context-1 model on vast.ai. See the adjacent `.md` for the runbook. |

```bash
cd ~/Projects/nexus/nexus-memory

# One-shot promoter
.venv/bin/python scripts/promoter.py --incremental

# Index a new project codebase (substitute your project name;
# the script in this repo is index_negotiagent.py)
.venv/bin/python scripts/index_<project>.py
```

## MCP server registration

Three MCP servers ship with Nexus. Register each once with the Claude Code CLI:

```bash
# Nexus MCP — 12 compound tools (chairman level)
claude mcp add nexus -- \
  ~/Projects/nexus/nexus-mcp/.venv/bin/python -m nexus_mcp.server

# Context-1 MCP — 3 retrieval tools
claude mcp add context1 -- python -m context1.mcp_server

# MemPalace MCP — 25 palace + KG tools
claude mcp add mempalace -- python -m mempalace.mcp_server
```

The Context-1 and MemPalace registration commands assume you run them from inside `~/Projects/nexus/nexus-memory/` (so the venv-installed packages are importable).

## Memory ingest hook

`~/.dotfiles/claude/scripts/memory_ingest.py` is the session-ingest hook wired into `~/.claude/settings.json` (`SessionEnd` / `Stop`). It can also be run manually for a full backfill.

```bash
# Backfill everything local
python ~/.dotfiles/claude/scripts/memory_ingest.py --all

# Latest session only (the hook's default invocation)
python ~/.dotfiles/claude/scripts/memory_ingest.py --latest-session
```

See [Ingestion](../concepts/ingestion.md) for the full ingestion pipeline.

## Shared-skills scripts

The `~/Projects/nexus/shared-skills/scripts/` directory ships three shell helpers that company repos invoke from their skills:

| Script | Purpose |
|---|---|
| `nexus-commit.sh` | Standardised commit wrapper used by company skill flows. |
| `nexus-record-branch.sh` | Record the current branch in a way the heartbeat can correlate to a ticket. |
| `nexus-verify.sh` | Pre-merge verification gate. |

## systemd units (operator-facing)

Not CLIs in the strict sense, but the operator-facing way to start and stop the core services. Units split across two scopes — see [File Locations → systemd units](file-locations.md#systemd-units) for the full table.

**System units** (`/etc/systemd/system/`, invoked with `sudo systemctl …`):

| Unit | Drives |
|---|---|
| `paperclip.service` | The Paperclip server (`paperclipai`) on `:3100`. |
| `meridian.service` | Meridian LLM proxy on `:3456`. |
| `chromadb.service` | ChromaDB vector store on `:8101`. |
| `mempalace-api.service` | MemPalace REST API on `:8102`. |

**User units** (`~/.config/systemd/user/`, invoked with `systemctl --user …`):

| Unit | Drives |
|---|---|
| `nexus-heartbeat.service` + `.timer` | `scripts/company_heartbeat.py`, fired every 5 minutes. |
| `mempalace-promoter.service` + `.timer` | `scripts/promoter.py --incremental`, fired every 15 minutes. |
| `nexus-memory-backup.service` + `.timer` | Daily hot snapshot to S3 at 03:00 via `scripts/s3_backup/backup_to_s3.sh`. |

```bash
# Status — system unit
sudo systemctl status paperclip

# Status — user timer
systemctl --user status nexus-heartbeat.timer
systemctl --user list-timers

# Manual fire — user timer's underlying service
systemctl --user start nexus-heartbeat.service

# Logs — system unit
sudo journalctl -u paperclip -f

# Logs — user units
journalctl --user -u nexus-heartbeat -f
journalctl --user -u mempalace-promoter -n 100
```

## See also

- [Paperclip component](../components/paperclip.md) — what `paperclipai` runs
- [Nexus Memory component](../components/nexus-memory.md) — context for the memory scripts
- [Nexus MCP component](../components/nexus-mcp.md) — the 12 compound tools `nexus-mcp` exposes
- [File Locations](file-locations.md) — where these scripts read and write
- [Environment Variables](env-vars.md) — what these scripts honour
- [Heartbeat concept](../concepts/heartbeat.md) — the lifecycle `company_heartbeat.py` implements
