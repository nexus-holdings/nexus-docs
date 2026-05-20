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
| 1 | PostgreSQL | systemd | 5432 | — | `nexus_metrics` DB for performance data |
| 2 | Ollama (or other LLM) | systemd | 11434 | — | Local LLM backend |
| 3 | Meridian | systemd | 3456 | — | API proxy (e.g., Claude → local) |
| 4 | Paperclip | systemd | 3100 | Meridian | Orchestration server + UI |
| 5 | Cockpit *(archived; optional)* | manual (npm) | 3000 | Paperclip, Postgres | Dashboard — pending rewrite as a plugin |
| 6 | Heartbeat | systemd timer | — | Paperclip, Postgres | Ticket dispatch worker |
| 7 | ChromaDB | manual / systemd | 8101 | — | Vector DB (Context-1) |
| 8 | MemPalace API | manual / systemd | 8102 | ChromaDB | Memory REST surface |

All services should bind to `127.0.0.1` for local-only use. Cockpit binds `0.0.0.0:3000` for browser access from other devices.

## Pre-flight

Confirm the env file points at an existing Paperclip company:

```bash
cat "$NEXUS_ROOT/nexus-core/.env"
```

`NEXUS_PAPERCLIP_COMPANY_ID` must match an existing [Paperclip](../components/paperclip.md) company (you'll create one in step 4 if it's a fresh install). See [Nexus Core](../components/nexus-core.md) for what consumes this env file.

## PostgreSQL

```bash
sudo systemctl start postgresql                # boot the DB
psql -d nexus_metrics -tAc \
  'SELECT COUNT(*) FROM performance_records;'  # smoke-test: confirms DB + role + table exist
```

The query should return an integer (probably `0` on a fresh install). If it errors, the DB or role is missing — fix that first.

## Ollama (or your chosen LLM backend)

```bash
sudo systemctl start ollama                                  # local LLM daemon
curl -s http://127.0.0.1:11434/api/tags \
  | python3 -m json.tool | head -20                          # list installed models
```

You should see a JSON list of installed models.

## Meridian (API proxy)

Paperclip calls the LLM via `ANTHROPIC_BASE_URL=http://127.0.0.1:3456`. Start Meridian **before** Paperclip.

```bash
sudo systemctl start meridian                                                       # boot the proxy
curl -s -o /dev/null -w "meridian: HTTP %{http_code}\n" http://127.0.0.1:3456/      # expect HTTP 200
```

HTTP 200 means the proxy is answering.

## Paperclip

```bash
sudo systemctl start paperclip                                          # boot the orchestrator
sleep 3                                                                 # give it a moment to bind :3100
curl -s http://127.0.0.1:3100/api/health | python3 -m json.tool         # expect status: ok
curl -s http://127.0.0.1:3100/api/companies | python3 -m json.tool      # list companies (may be empty)
```

Health should report `"status": "ok"`. The company list should contain at least one company (or be empty on first run — you'll create one shortly).

Logs: `journalctl -u paperclip -f`

## Cockpit (dashboard)

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

## ChromaDB + MemPalace API

These power [Nexus Memory](../components/nexus-memory.md).

```bash
cd "$NEXUS_ROOT/nexus-memory"

# ChromaDB on 8101
nohup .venv/bin/chroma run --host 0.0.0.0 --port 8101 \
    --path "$NEXUS_ROOT/nexus-memory/data/chromadb" \
    > data/chromadb.log 2>&1 &

# MemPalace REST API on 8102 — with GPU bge-m3 for the promoter path
BGE_M3_DEVICE=cuda nohup .venv/bin/python api/server.py \
    > /tmp/nexus-memory-api.log 2>&1 &

sleep 3
curl -s http://127.0.0.1:8101/api/v2/heartbeat                       # ChromaDB liveness
curl -s http://127.0.0.1:8102/health                                 # MemPalace liveness
```

Both should return health-ok payloads.

## Heartbeat

Without the heartbeat, tickets sit in `backlog` until manually advanced. Start it to enable automatic dispatch.

Install the systemd timer and enable it to tick every 5 minutes:

```bash
systemctl --user enable --now nexus-heartbeat.timer                  # install + start the 5-min timer
systemctl --user list-timers nexus-heartbeat.timer                   # confirm the next-fire time
```

Or run it manually for development:

```bash
cd "$NEXUS_ROOT/nexus-core"
nohup uv run python scripts/company_heartbeat.py \
    > /tmp/nexus-heartbeat.log 2>&1 &
```

## Verify the whole stack

```bash
curl -s http://127.0.0.1:3100/api/health         # Paperclip
curl -s http://127.0.0.1:3000                    # Cockpit (HTML)
curl -s http://127.0.0.1:8101/api/v2/heartbeat   # ChromaDB
curl -s http://127.0.0.1:8102/health             # MemPalace API
```

If all four respond, you're ready for the [First Ticket](first-ticket.md) walkthrough.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Paperclip health fails | Meridian not up | Start Meridian first; check `journalctl -u meridian` |
| Cockpit 502 / blank page | Paperclip down or wrong port | `curl :3100/api/health`; rebuild Cockpit if needed |
| Heartbeat says "memory infra unhealthy" | ChromaDB or MemPalace API down | Start them; check ports 8101 / 8102 |
| Sessions not landing in MemPalace | Stop hook not installed | Check `~/.claude/settings.json` has the `Stop` hook entry |
| Promoter on CPU (slow, no GPU fan) | `BGE_M3_DEVICE` not set | Restart API with `BGE_M3_DEVICE=cuda` |

See also: [CLI Commands](../reference/cli-commands.md), [Environment Variables](../reference/env-vars.md), [Paperclip](../components/paperclip.md), [Nexus Core](../components/nexus-core.md).
