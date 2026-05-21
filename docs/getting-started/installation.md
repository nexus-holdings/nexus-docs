# Installation

Get the full Nexus stack running on a single host. This walks through bringing up the core services in order — each later service depends on the earlier ones.

!!! info "Prerequisites"
    - Linux (tested on Ubuntu 24.04)
    - Python 3.12+
    - Node 22+
    - PostgreSQL 14+
    - `uv` (Python tooling)
    - A working Claude or local LLM endpoint

## Service map

| # | Service | Type | Port | Depends on | Purpose |
|---|---|---|---|---|---|
| 1 | PostgreSQL | systemd (system) | 5432 | — | `nexus_metrics` DB for performance data |
| 2 | Ollama (or other LLM) | systemd (system) | 11434 | — | Local LLM backend |
| 3 | Meridian | systemd (system) | 3456 | — | API proxy (e.g., Claude → local) |
| 4 | Paperclip | systemd (system) | 3100 | Meridian | Orchestration server + UI |
| 5 | ChromaDB | systemd (system) | 8101 | — | Vector DB (Context-1) |
| 6 | MemPalace API | systemd (system) | 8102 | ChromaDB | Memory REST surface |
| 7 | Cockpit *(archived; optional)* | manual (npm) | 3000 | Paperclip, Postgres | Dashboard — pending rewrite as a plugin |
| 8 | Heartbeat | systemd timer (user) | — | Paperclip, Postgres | Ticket dispatch worker |
| 9 | Promoter | systemd timer (user) | — | MemPalace API | MemPalace → Context-1 sync (every 15 min) |
| 10 | Memory backup | systemd timer (user) | — | MemPalace API | Daily hot-snapshot to S3 |

All services bind to `127.0.0.1` for local-only use. Cockpit binds `0.0.0.0:3000` for browser access from other devices.

**Two unit scopes are in play:**

- **System-level units** (`/etc/systemd/system/*.service`) — boot at host startup, managed with `sudo systemctl`. Used for long-running services (Paperclip, Meridian, ChromaDB, MemPalace API).
- **User-level units** (`~/.config/systemd/user/*.{service,timer}`) — bound to your login, managed with `systemctl --user`. Used for the recurring timer-based tasks that need access to your `uv` environment (heartbeat, promoter, backup).

## Quick start: one-command boot

If the host already has the unit files installed, bring everything up with:

```bash
cd ~/Projects/nexus
./nexus-up.sh                                # idempotent; safe to re-run
```

This script handles all the service starts in dependency order and prints a summary at the end. The rest of this page is the manual walkthrough — useful for first-time installs or debugging individual services.

## Pre-flight

Confirm the env file points at an existing Paperclip company:

```bash
cat "$NEXUS_ROOT/nexus-core/.env"
```

`NEXUS_PAPERCLIP_COMPANY_ID` must match an existing [Paperclip](../components/paperclip.md) company (you'll create one in step 4 if it's a fresh install). See [Nexus Core](../components/nexus-core.md) for what consumes this env file.

## 1 — PostgreSQL

```bash
sudo systemctl start postgresql                # boot the DB
psql -d nexus_metrics -tAc \
  'SELECT COUNT(*) FROM performance_records;'  # smoke-test: confirms DB + role + table exist
```

The query should return an integer (probably `0` on a fresh install). If it errors, the DB or role is missing — fix that first.

## 2 — Ollama (or your chosen LLM backend)

```bash
sudo systemctl start ollama                                  # local LLM daemon
curl -s http://127.0.0.1:11434/api/tags \
  | python3 -m json.tool | head -20                          # list installed models
```

You should see a JSON list of installed models.

## 3 — Meridian (API proxy)

Paperclip calls the LLM via `ANTHROPIC_BASE_URL=http://127.0.0.1:3456`. Start Meridian **before** Paperclip.

```bash
sudo systemctl start meridian                                                       # boot the proxy
curl -s -o /dev/null -w "meridian: HTTP %{http_code}\n" http://127.0.0.1:3456/      # expect HTTP 200
```

HTTP 200 means the proxy is answering.

## 4 — Paperclip

```bash
sudo systemctl start paperclip                                          # boot the orchestrator
sleep 3                                                                 # give it a moment to bind :3100
curl -s http://127.0.0.1:3100/api/health | python3 -m json.tool         # expect status: ok
curl -s http://127.0.0.1:3100/api/companies | python3 -m json.tool      # list companies (may be empty)
```

Health should report `"status": "ok"`. The company list should contain at least one company (or be empty on first run — you'll create one shortly).

Logs: `journalctl -u paperclip -f`

## 5 — ChromaDB + MemPalace API

These power [Nexus Memory](../components/nexus-memory.md). MemPalace API requires ChromaDB, so start ChromaDB first (the unit declares the dependency, so `start mempalace-api` will pull ChromaDB up too).

```bash
sudo systemctl start chromadb                                # vector store on :8101
sudo systemctl start mempalace-api                           # REST surface on :8102

sleep 3
curl -s http://127.0.0.1:8101/api/v2/heartbeat               # ChromaDB liveness
curl -s http://127.0.0.1:8102/health                         # MemPalace liveness
```

Both should return health-ok payloads.

Logs: `journalctl -u chromadb -f` and `journalctl -u mempalace-api -f`.

## 6 — Cockpit (dashboard)

!!! warning "Deprecated — pending rewrite"
    The original standalone Cockpit (Next.js, port 3000) is **archived**. A replacement is being built as a plugin rendered inside the governance company. The steps below still work against the archived checkout if it exists, but you can skip this section on a fresh install — nothing downstream depends on it.

```bash
cd "$NEXUS_ROOT/cockpit"
nohup npm run start > /tmp/cockpit.log 2>&1 &                                     # launch Next.js in background
sleep 4                                                                            # wait for the server to come up
curl -s -o /dev/null -w "cockpit: HTTP %{http_code}\n" http://localhost:3000      # expect HTTP 200
```

If Cockpit has never been built on this checkout, run `npm run build` first.

Open in browser: [http://localhost:3000](http://localhost:3000)

## 7 — Recurring tasks (heartbeat + promoter + backup)

These are **user-level systemd timers** — they live in `~/.config/systemd/user/` and tick on your login session. Install + enable each one:

```bash
# Heartbeat: pick up backlog tickets every 5 min
systemctl --user enable --now nexus-heartbeat.timer

# Promoter: sync MemPalace → Context-1 every 15 min
systemctl --user enable --now mempalace-promoter.timer

# Backup: hot-snapshot MemPalace + Context-1 to S3 daily
systemctl --user enable --now nexus-memory-backup.timer

# Confirm next-fire times
systemctl --user list-timers
```

Without the heartbeat, tickets sit in `backlog` until manually advanced.

For manual / dev runs:

```bash
cd "$NEXUS_ROOT/nexus-core"
nohup uv run python scripts/company_heartbeat.py \
    > /tmp/nexus-heartbeat.log 2>&1 &
```

## Installing the unit files (first-time setup)

The system-level unit files (Paperclip, Meridian, ChromaDB, MemPalace API) live as templates in source. If they aren't installed on this host yet, run the install scripts:

```bash
# Paperclip + Meridian
cd "$NEXUS_ROOT/nexus-core/deploy/systemd"
./install.sh                                  # copies units to /etc/systemd/system/

# ChromaDB + MemPalace API
cd "$NEXUS_ROOT/nexus-memory/deploy/systemd"
./install.sh                                  # copies units to /etc/systemd/system/
```

The user-level timer units (heartbeat, promoter, backup) are installed when you first run `systemctl --user enable --now <name>.timer` against the unit files in `~/.config/systemd/user/`.

## Verify the whole stack

```bash
curl -s http://127.0.0.1:3100/api/health         # Paperclip
curl -s http://127.0.0.1:3000                    # Cockpit (HTML, if running)
curl -s http://127.0.0.1:8101/api/v2/heartbeat   # ChromaDB
curl -s http://127.0.0.1:8102/health             # MemPalace API
systemctl --user list-timers --no-pager          # heartbeat + promoter + backup
```

If all four HTTP probes respond and the three timers show next-fire times, you're ready for the [First Ticket](first-ticket.md) walkthrough.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Unit X.service not found` | Unit file not installed | Run the `install.sh` in `nexus-core/deploy/systemd/` or `nexus-memory/deploy/systemd/` |
| Paperclip health fails | Meridian not up | Start Meridian first; check `journalctl -u meridian` |
| Cockpit 502 / blank page | Paperclip down or wrong port | `curl :3100/api/health`; rebuild Cockpit if needed |
| Heartbeat says "memory infra unhealthy" | ChromaDB or MemPalace API down | `sudo systemctl status chromadb mempalace-api` |
| Sessions not landing in MemPalace | Stop hook not installed | Check `~/.claude/settings.json` has the `Stop` hook entry |
| Promoter on CPU (slow, no GPU fan) | `BGE_M3_DEVICE` not set | Edit the unit's `Environment=BGE_M3_DEVICE=cuda` and `sudo systemctl daemon-reload && sudo systemctl restart mempalace-api` |
| `sudo systemctl daemon-reload` complains about a service | Unit file syntax error | `systemd-analyze verify /etc/systemd/system/<name>.service` |

See also: [CLI Commands](../reference/cli-commands.md), [Environment Variables](../reference/env-vars.md), [Paperclip](../components/paperclip.md), [Nexus Core](../components/nexus-core.md).
