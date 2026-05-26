# File Locations

<p class="lede">A single map of the files and directories Nexus reads, writes, and depends on — data dirs, configs, logs, catalogs, state. Everything below has a verified location; the column "owned by" names the component that writes there.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-20</span>
  <span>Owner: Platform</span>
</div>

## Repos (source roots)

All Nexus repos sit under `~/Projects/nexus/`. The canonical set:

| Path | What it is |
|---|---|
| `~/Projects/nexus/paperclip/` | Paperclip's data directory (the `paperclipai` npm package is installed globally, not here). |
| `~/Projects/nexus/nexus-core/` | Heartbeat, ops monitor, provisioning, chairman CLI. Python. |
| `~/Projects/nexus/nexus-memory/` | REST API, Context-1, promoter, MemPalace scripts. Python. |
| `~/Projects/nexus/nexus-mcp/` | Chairman-level MCP server. Python. |
| `~/Projects/nexus/cockpit/` | Next.js dashboard. TypeScript. |
| `~/Projects/nexus/paperclip-plugin-acp/` | ACP session lifecycle plugin. TypeScript. |
| `~/Projects/nexus/paperclip-plugin-craft-dispatch/` | Domain → craft dispatch + flow-back. TypeScript. |
| `~/Projects/nexus/paperclip-plugin-contracts/` | Contracts lifecycle. TypeScript. |
| `~/Projects/nexus/paperclip-plugin-memory/` | Memory bridge plugin. TypeScript. |
| `~/Projects/nexus/agent-catalog/` | Agent + team YAML definitions. |
| `~/Projects/nexus/routine-catalog/` | Routine YAML definitions. |
| `~/Projects/nexus/eval-registry/` | Eval rubrics + specs. |
| `~/Projects/nexus/shared-skills/` | Shared skill markdown + helper scripts. |
| `~/Projects/nexus/company-template/` | Template a new company is provisioned from. |
| `~/Projects/nexus/self-improvement/` | Self-improvement company sources. |
| `~/Projects/nexus/site/` | This documentation site (MkDocs Material). |

When you see `$NEXUS_ROOT` referenced elsewhere, it points at `~/Projects/nexus/`.

## Data directories

Where the runtime state lives.

| Path | Purpose | Owned by |
|---|---|---|
| `~/Projects/nexus/paperclip/data/instances/default/` | Active Paperclip instance root. | Paperclip |
| `~/Projects/nexus/paperclip/data/instances/default/db/` | Embedded Postgres data dir (port `54329`). | Paperclip |
| `~/Projects/nexus/paperclip/data/instances/default/data/storage/` | Local-disk object storage for attachments. | Paperclip |
| `~/Projects/nexus/paperclip/data/instances/default/data/backups/` | Hourly Postgres backups (30-day retention by default). | Paperclip |
| `~/Projects/nexus/paperclip/data/instances/default/companies/` | Per-company workspace dirs. | Paperclip |
| `~/.mempalace/palace/` | MemPalace palace data — `chroma.sqlite3` + HNSW segments. | MemPalace |
| `~/.mempalace/knowledge_graph.sqlite3` | Knowledge-graph store (subject/predicate/object triples + temporal validity). | MemPalace |
| `~/.mempalace/wal/` | MemPalace write-ahead log. | MemPalace |
| `~/.mempalace/config.json` | MemPalace local config (palace path override). | MemPalace |
| `~/Projects/nexus/nexus-memory/data/chromadb/` | Context-1 ChromaDB persistence (one collection per wing under `mp-<wing>`). | Context-1 |
| `nexus_metrics` (Postgres DB) | Performance records + agent KPIs. DSN from `NEXUS_METRICS_DB`. | Cockpit + ACP plugin + ingest scripts |

## Config files

| Path | Purpose |
|---|---|
| `~/Projects/nexus/paperclip/data/instances/default/config.json` | The active Paperclip config — DB mode, bind host/port, storage provider, logging dir, secrets provider. |
| `~/Projects/nexus/paperclip/data/instances/default/secrets/master.key` | Local-encrypted secrets key. Don't commit. |
| `~/Projects/nexus/nexus-core/.env` | Nexus Core env file — metrics DSN, Paperclip API base, catalog paths, GitHub org. |
| `~/Projects/nexus/cockpit/.env.local` | Cockpit env file (gitignored). |
| `~/Projects/nexus/cockpit/.env.example` | Reference template for the cockpit env. |
| `~/Projects/nexus/company-template/templates/.env.example*` | Templates copied into newly-provisioned companies. |
| `~/.claude/settings.json` | Claude Code harness settings — hook config (e.g. `SessionEnd` → `memory_ingest.py`), MCP servers. |
| `~/.claude/CLAUDE.md` | Global Claude Code preferences. |
| `/etc/systemd/system/*.service` | systemd **system** unit files for the always-on Nexus services (see below). |
| `~/.config/systemd/user/*.service` + `*.timer` | systemd **user** unit files for the timer-driven Nexus services (see below). |

## systemd units

Nexus services split across two systemd scopes — pick the right verb prefix or `systemctl` will silently not find the unit.

**System units** (`/etc/systemd/system/`, invoked with `sudo systemctl …`):

| Unit | Drives |
|---|---|
| `paperclip.service` | The Paperclip server. |
| `meridian.service` | Meridian LLM proxy on `:3456`. |
| `chromadb.service` | ChromaDB vector store on `:8101`. |
| `mempalace-api.service` | MemPalace REST API on `:8102`. |

**User units** (`~/.config/systemd/user/`, invoked with `systemctl --user …`):

| Unit | Drives |
|---|---|
| `nexus-heartbeat.service` + `nexus-heartbeat.timer` | `nexus-core/scripts/company_heartbeat.py` every 5 min. |
| `mempalace-promoter.service` + `mempalace-promoter.timer` | `nexus-memory/scripts/promoter.py --incremental` every 15 min. |
| `nexus-memory-backup.service` + `nexus-memory-backup.timer` | Daily S3 backup at 03:00. |
| `gemma-llama.service` | Optional Gemma 4 26B local inference server (`llama.cpp` on `:8000`). |
| `claude-state-sync.service` + `claude-state-sync.timer` | Claude Code state sync (off-repo). |

See [Installation](../getting-started/installation.md) for the install + enable flow.

## Logs

| Where | What | How to tail |
|---|---|---|
| `journalctl -u paperclip` | Paperclip server logs (system unit). | `sudo journalctl -u paperclip -f` |
| `journalctl -u meridian` | Meridian proxy logs (system unit). | `sudo journalctl -u meridian -f` |
| `journalctl -u chromadb` | ChromaDB logs (system unit). | `sudo journalctl -u chromadb -f` |
| `journalctl -u mempalace-api` | MemPalace REST API logs (system unit). | `sudo journalctl -u mempalace-api -f` |
| `journalctl --user -u nexus-heartbeat` | Heartbeat run output. | `journalctl --user -u nexus-heartbeat -f` |
| `journalctl --user -u mempalace-promoter` | Promoter output (also appended to `/tmp/promoter.log`). | `journalctl --user -u mempalace-promoter -n 100` |
| `journalctl --user -u nexus-memory-backup` | Backup job output (also `/tmp/backup.log`). | `journalctl --user -u nexus-memory-backup` |
| `~/Projects/nexus/paperclip/data/instances/default/logs/server.log` | Paperclip file log (per `config.json` `logging.logDir`). | `tail -f` |
| `~/Projects/nexus/nexus-memory/data/chromadb.log` | ChromaDB server log. | `tail -f` |
| `/tmp/promoter.log` | Append-only log from the promoter timer (mirrors journal). | — |
| `/tmp/backup.log` | Append-only log from the backup timer. | — |

## Catalogs

Source-of-truth data the platform reads at runtime or deploy time.

| Path | Contents | Consumed by |
|---|---|---|
| `~/Projects/nexus/agent-catalog/agents/*.yaml` | One YAML per agent (prompt + model + tools). | `nexus_deploy_cli.py agents`, `setup_csuite.py`. |
| `~/Projects/nexus/agent-catalog/teams/*.yaml` | Team definitions — which agents form a team. | `nexus_deploy_cli.py team`. |
| `~/Projects/nexus/routine-catalog/routines/*.yaml` | Cron-style routine specs. | Paperclip routine sync, manual triggers. |
| `~/Projects/nexus/eval-registry/specs/<dimension>/` | Eval rubrics — `code-quality`, `design-adherence`, `review-efficiency`, `run-quality`, `security-scan`, etc. | The eval plugin and ACP merge-agent. |
| `~/Projects/nexus/eval-registry/rubrics/` | Versioned rubric files referenced from `eval_versions` in performance records. | The metrics pipeline. |
| `~/Projects/nexus/shared-skills/<skill>/` | Shared skill markdown bundles. | `nexus_deploy_cli.py skills`. |
| `~/Projects/nexus/shared-skills/scripts/*.sh` | Shell helpers (commit/branch/verify) consumed by skill flows. | Company-side skills. |

## State files

Transient state outside the databases.

| Path | Purpose | Owned by |
|---|---|---|
| `/tmp/promoter-claude.lock` | Per-wing PID lock — prevents concurrent promoter runs on the same wing. | promoter |
| `/tmp/promoter.log` | Append-only promoter log. | promoter unit |
| `/tmp/backup.log` | Append-only backup log. | backup unit |
| `/tmp/nexus-heartbeat.json` | Optional heartbeat state file (override path with `HEARTBEAT_STATE_PATH`). | heartbeat |
| `/tmp/paperclip-attachments/` | Temp dir for in-flight attachment uploads. | Paperclip |
| `/tmp/reviewer-verdict-<session_id>.attempted` | Reviewer-verdict idempotence marker (set when the ACP review hook posts a verdict). | ACP plugin |
| `~/.backups/<YYYY-MM-DD-HHMMSS>[-<label>]/` | Cold-snapshot output from `nexus-memory/scripts/cold_snapshot.sh`. Contains palace + KG + WAL + config + Context-1 Chroma data. | cold-snapshot script |

## Session transcripts

Claude Code sessions get persisted under `~/.claude/projects/<encoded-cwd>/`. The encoding swaps `/` for `-`, so a session run from `~/Projects/nexus/` lands at `~/.claude/projects/-home-ian-Projects-nexus/`. Inside each project directory:

| Path | What |
|---|---|
| `~/.claude/projects/<project>/*.jsonl` | One file per session — the raw transcript. The `memory_ingest.py` hook ingests these. |
| `~/.claude/projects/<project>/memory/*.md` | Auto-memory entries. Each `.md` is ingested as one drawer on write. |

## Binaries on `PATH`

For completeness, the binaries CLI users hit by name:

| Binary | Resolved path |
|---|---|
| `paperclipai` | `~/.npm-global/bin/paperclipai` |
| `mempalace` | `~/Projects/nexus/nexus-memory/.venv/bin/mempalace` |
| `nexus-mcp` | `~/Projects/nexus/nexus-mcp/.venv/bin/nexus-mcp` |
| `uv` | `~/.local/bin/uv` |

## See also

- [Installation](../getting-started/installation.md) — the bootstrap flow that lays these down
- [CLI Commands](cli-commands.md) — the scripts and binaries that read these paths
- [Environment Variables](env-vars.md) — overrides for the locations on this page
- [Memory layer](../architecture/memory-layer.md) — why MemPalace and Context-1 are split across two of these paths
- [Paperclip component](../components/paperclip.md) — what lives under `paperclip/data/`
