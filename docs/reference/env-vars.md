# Environment Variables

<p class="lede">The full catalogue of environment variables Nexus reads at runtime, grouped by the component that consumes them. Defaults come straight from source; every entry has a verified read site. Variables not listed here are not consumed by Nexus.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-20</span>
  <span>Owner: Platform</span>
</div>

## How to set these

Two canonical locations:

- **Per-process**: `~/Projects/nexus/nexus-core/.env` (for nexus-core scripts) and `~/Projects/nexus/cockpit/.env.local` (for the dashboard). See `cockpit/.env.example` for the cockpit template.
- **Per-systemd-unit**: each unit in `~/.config/systemd/user/` sets its own `Environment=` lines. The heartbeat unit, for instance, pins `NEXUS_MAX_LIVE_WORKERS=1` regardless of what the user shell has set.

Plugin processes inherit the environment of the Paperclip server that launched them.

!!! note "Most variables are optional"
    Every variable on this page is read **with a sensible default** — you only need to set the ones whose defaults you want to override. A fresh `.env` file with no entries works for most installations; explicit values are mainly for environment-specific overrides (different ports, custom paths, ops kill-switches). Entries marked _(required)_ or _(unset)_ are the exceptions.

## Nexus Core (heartbeat, ops monitor, scripts)

Read by `~/Projects/nexus/nexus-core/scripts/company_heartbeat.py` and friends.

| Variable | Default | Purpose |
|---|---|---|
| `NEXUS_MAX_LIVE_WORKERS` | `1` | Global cap on concurrent heartbeat-spawned workers. Phase-1 conservative default. |
| `NEXUS_REQUIRE_MEMORY_INFRA` | `true` | When `true`, heartbeat refuses to spawn workers if the memory stack is unhealthy. |
| `NEXUS_IDLE_ESCALATION_DISABLED` | `false` | When `true`, suppresses heartbeat idle-pattern escalation. Set in the unit file. |
| `OPS_STUCK_THRESHOLD_MINUTES` | `15` | `ops_monitor.py` — ticket in `in_progress` longer than this is flagged stuck. |
| `OPS_HEARTBEAT_STALE_MINUTES` | `12` | `ops_monitor.py` — alert if the heartbeat log file hasn't been written in this window. |
| `OPS_SESSION_OVERUSE_THRESHOLD` | `18` | `ops_monitor.py` — alert when total `in_progress` tickets exceed this. |
| `PAPERCLIP_TASK_ID` | _(unset)_ | Read by `nexus/execution/skill_flags.py` to bias skill-priority resolution when the worker is running inside a Paperclip task. |

### Nexus Core `.env`

Read indirectly by ancillary scripts and provisioning tools. The live template:

| Variable | Example | Purpose |
|---|---|---|
| `NEXUS_METRICS_DB` | `postgresql:///nexus_metrics` | Postgres DSN for the metrics warehouse. Also read by the cockpit and the ACP plugin. |
| `NEXUS_PAPERCLIP_API_BASE` | `http://127.0.0.1:3100/api` | Override of `PAPERCLIP_API_BASE` for nexus-core tooling. |
| `NEXUS_PAPERCLIP_COMPANY_ID` | `<uuid>` | Default company for chairman/meeting flows. |
| `NEXUS_AGENT_CATALOG_PATH` | `~/Projects/nexus/agent-catalog/agents` | Where `nexus deploy agents` reads agent definitions. |
| `NEXUS_AGENT_CATALOG_TEAMS_PATH` | `~/Projects/nexus/agent-catalog/teams` | Team definitions. |
| `NEXUS_SHARED_SKILLS_DIR` | `~/Projects/nexus/shared-skills` | Skill source. |
| `NEXUS_COMPANY_TEMPLATE_DIR` | `~/Projects/nexus/company-template` | Used by `provision_company.py` and `find_template_drift`. |
| `NEXUS_GITHUB_ORG` | `nexus-holdings` | Org used when provisioning new company repos. |

## Cockpit (Next.js dashboard)

Read by `~/Projects/nexus/cockpit/src/lib/*.ts`. Defaults from `cockpit/.env.example`.

| Variable | Default | Purpose |
|---|---|---|
| `NEXUS_METRICS_DB` | `postgresql://localhost:5432/nexus_metrics` | Metrics warehouse DSN. |
| `NEXUS_DB_POOL_MIN` | `1` | Min Postgres pool size. |
| `NEXUS_DB_POOL_MAX` | `10` | Max Postgres pool size. |
| `PAPERCLIP_API_BASE` | `http://127.0.0.1:3100/api` | Paperclip REST base for server-side fetches. |
| `PAPERCLIP_COMPANY_ID` | _(per-deployment)_ | Default company the dashboard scopes to. |
| `NEXUS_PAPERCLIP_POLL_INTERVAL` | `10000` (ms) | How often the cockpit polls Paperclip for ticket changes. |
| `NEXT_PUBLIC_PAPERCLIP_UI_BASE` | `http://localhost:3100` | Browser-visible Paperclip UI URL (used by the embed). |
| `NEXUS_WS_PORT` | `3001` | Cockpit metrics websocket port. |
| `NEXUS_WS_POLL_INTERVAL` | `5000` (ms) | Heartbeat poll interval over WS. |
| `NEXT_PUBLIC_NEXUS_WS_URL` | _(derived)_ | Browser-visible WS URL; defaults from `NEXUS_WS_PORT`. |
| `NEXUS_SUBSCRIPTION_WEEKLY_TOKEN_ALLOWANCE` | `50000000` | Subscription budget rails. |
| `NEXUS_SUBSCRIPTION_MONTHLY_TOKEN_ALLOWANCE` | `200000000` | Subscription budget rails. |
| `NEXUS_SUBSCRIPTION_UTILIZATION_WARN` | `0.70` | Throttle band start (warn). |
| `NEXUS_SUBSCRIPTION_UTILIZATION_THROTTLE` | `0.85` | Throttle band middle. |
| `NEXUS_SUBSCRIPTION_UTILIZATION_PAUSE` | `0.95` | Throttle band end (pause). |
| `NEXUS_SESSIONS_TOTAL` | `20` | Total session budget. |
| `NEXUS_SESSIONS_PARENT_RESERVED` | `2` | Sessions reserved for the parent agent. |
| `NEXUS_DEFAULT_SESSION_CAP` | `2` | Default per-company session cap. |
| `NEXUS_SCHEDULER_FLOW_SNAPSHOT_PATH` | _(repo-relative)_ | Where the cockpit reads the scheduler-flow snapshot from. |
| `NEXUS_CORE_ROOT` | `~/Projects/nexus/nexus-core` | Used by the cockpit to locate nexus-core artifacts. |
| `HEARTBEAT_STATE_PATH` | `/tmp/nexus-heartbeat.json` | Optional override for the heartbeat state file the cockpit reads. |
| `TAILSCALE_ALLOWED_CIDRS` | _(unset)_ | If set, restricts dashboard access to these CIDR ranges. |
| `TAILSCALE_AUTH_BYPASS` | `true` | If `true`, skips Tailscale-based auth (local-dev default). |

## Nexus Memory (REST API + Context-1 + promoter)

Read by `~/Projects/nexus/nexus-memory/api/server.py`, `context1/*.py`, and `scripts/promoter.py`.

| Variable | Default | Purpose |
|---|---|---|
| `MEMORY_API_PORT` | `8102` | Bind port for the FastAPI sidecar. |
| `MEMPALACE_BIN` | `<repo>/.venv/bin/mempalace` | Path the API shells out to for `wake-up`. |
| `MEMPALACE_PATH` | `~/.mempalace/palace` | Palace data directory. Read by the promoter and the `mempalace` package. |
| `CHROMA_HOST` | `localhost` (or `127.0.0.1` for indexers) | Where the Context-1 ChromaDB server lives. |
| `CHROMA_PORT` | `8101` | ChromaDB port. |
| `CONTEXT1_BASE_URL` | `http://127.0.0.1:8003/v1` | LLM endpoint used by the Context-1 agentic retrieval loop. |
| `CONTEXT1_MODEL` | `~/models/context-1-mxfp4` | Model path for the agentic loop. |
| `BGE_M3_DEVICE` | `cpu` | Embedder device — set to `cuda` for GPU. |

## Plugins (Paperclip extensions)

Read by the four shipped plugins in `~/Projects/nexus/paperclip-plugin-*/src/constants.ts`.

| Variable | Default | Read by |
|---|---|---|
| `PAPERCLIP_API_BASE` | `http://127.0.0.1:3100/api` | All plugins. |
| `PAPERCLIP_DB` | `postgresql://paperclip:paperclip@127.0.0.1:54329/paperclip` | acp, craft-dispatch, contracts (direct SQL for `origin_kind`/`origin_id`). |
| `MEMORY_API_BASE` | `http://127.0.0.1:8102` | acp, craft-dispatch, memory plugin. |
| `MERIDIAN_BASE` | `http://127.0.0.1:3456` | acp plugin (Anthropic-proxy health check). |
| `NEXUS_METRICS_DB` | _(see above)_ | acp plugin (records performance metrics). |
| `ACP_MODEL_OVERRIDE` | _(unset)_ | acp plugin — ops kill-switch that pins every Claude dispatch to one model. See `acp/src/model-selection.ts`. |
| `MERGE_AGENT_DRY_RUN` | `false` | acp merge-agent — when `true`, simulates the merge without writing. |
| `DISPATCH_ENRICHMENT_ENABLED` | _(unset)_ | craft-dispatch — when not `false`, enriches dispatched tickets with memory context. |
| `HOME` | _(system)_ | acp spawn (inherited into spawned agent processes). |

## S3 backup Lambda

Read by `~/Projects/nexus/nexus-memory/scripts/s3_backup/lambda_function.py`.

| Variable | Default | Purpose |
|---|---|---|
| `AUTH_TOKEN` | `""` | Bearer token the Lambda compares against. |
| `BUCKET_NAME` | _(required)_ | S3 bucket for backup uploads. |
| `HOSTNAME_PFX` | `host` | Object-key prefix. |

## ChromaDB indexers (one-shot scripts)

Read by `~/Projects/nexus/nexus-memory/scripts/index_<project>.py` and `migrate_to_bgem3.py`.

| Variable | Default | Purpose |
|---|---|---|
| `CHROMA_HOST` | `127.0.0.1` | ChromaDB host. |
| `CHROMA_PORT` | `8101` | ChromaDB port. |
| `BGE_M3_DEVICE` | _(`cuda` in `migrate_to_bgem3.py`)_ | Embedder device. |

## Not currently consumed

The following appear in older docs or audits but are not read by any live source as of this writing. Listed so future readers don't search in vain:

- `NEXUS_PAPERCLIP_COMPANY_ID` lives in the `.env` template; the live cockpit reads `PAPERCLIP_COMPANY_ID` instead.
- `NEXUS_AGENT_CATALOG_PATH` / `NEXUS_COMPANY_TEMPLATE_REPO` exist in the `.env` template; live consumers prefer the catalog paths the `nexus deploy` CLI computes from `NEXUS_CORE_ROOT`.
- `ANTHROPIC_BASE_URL` is mentioned in older Paperclip systemd-unit examples but the live plugin code uses `MERIDIAN_BASE` instead. Set both during a transition window.
- `GITHUB_TOKEN` is read by tooling outside the four canonical Nexus repos (provisioning scripts that shell out to `gh`); not Nexus-owned.

## See also

- [Paperclip API](api-paperclip.md) — what `PAPERCLIP_API_BASE` points at
- [MemPalace API](api-mempalace.md) — what `MEMORY_API_BASE` points at
- [File Locations](file-locations.md) — paths these variables typically reference
- [Heartbeat concept](../concepts/heartbeat.md) — uses the `NEXUS_MAX_LIVE_WORKERS` / `OPS_*` knobs
- [Cockpit component](../components/cockpit.md) — consumer of most `NEXUS_*` cockpit vars
