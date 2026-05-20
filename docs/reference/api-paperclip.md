# Paperclip API

<p class="lede">Paperclip exposes a Linear-style REST surface on <code>http://127.0.0.1:3100/api/...</code> — loopback-only, JSON in/JSON out, no authentication beyond network reachability. Every entity Paperclip owns (companies, issues, projects, agents, plugins, routines) is reachable here. This page documents the endpoints that have been verified against a live <code>2026.403.0</code> instance and the plugin source.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-20</span>
  <span>Owner: Platform</span>
</div>

## What it owns

Paperclip is the source of truth for the platform-layer entity set. Other components query Paperclip rather than holding their own copies.

| Entity | Resource path | Notes |
|---|---|---|
| **Company** | `/api/companies/<id>` | A unit of work scope. See [Companies](../concepts/companies.md). |
| **Issue** | `/api/issues/<id>` | A ticket. See [Tickets](../concepts/tickets.md). Internally still called `issues`. |
| **Project** | `/api/companies/<id>/projects` | Sub-grouping of issues inside a company. |
| **Agent (member)** | `/api/companies/<id>/agents` | Configured worker — prompt + tools + model. |
| **Plugin** | `/api/plugins` | Registered Paperclip extension. |
| **Routine** | `/api/companies/<id>/routines` | Cron-style recurring task. |
| **Label** | `/api/companies/<id>/labels` | Company-scoped tag. |

## Endpoint reference

The paths below are exactly the ones called from `nexus-mcp/clients/paperclip.py` and the four shipped plugins (`paperclip-plugin-acp`, `paperclip-plugin-craft-dispatch`, `paperclip-plugin-contracts`, `paperclip-plugin-memory`). Every entry was probed against a running instance; status codes record live behaviour at the time of writing.

### Health

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/health` | Liveness + bootstrap status. Returns `{status, version, deploymentMode, deploymentExposure, authReady, bootstrapStatus, ...}`. |

```bash
curl -s http://127.0.0.1:3100/api/health
# {"status":"ok","version":"2026.403.0","deploymentMode":"local_trusted",...}
```

### Companies

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/companies` | List every company. |
| `GET` | `/api/companies/<company_id>` | Fetch one company by UUID. |
| `PATCH` | `/api/companies/<company_id>` | Update company fields (mutating). |

The list endpoint returns objects shaped like `{id, name, description, status, issuePrefix, issueCounter, budgetMonthlyCents, ...}`.

### Issues (tickets)

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/companies/<company_id>/issues` | List issues scoped to a company. |
| `POST` | `/api/companies/<company_id>/issues` | Create an issue in the company. Body: `{title, description, status, priority, projectId?}`. |
| `GET` | `/api/issues/<issue_id>` | Read one issue. |
| `PATCH` | `/api/issues/<issue_id>` | Patch issue fields — most commonly `{status}`. Used by craft-dispatch flow-back and the ACP webhook hooks to drive the state machine. |
| `GET` | `/api/issues/<issue_id>/comments` | List comments on an issue. |
| `POST` | `/api/issues/<issue_id>/comments` | Append a comment. Body: `{body}`. |
| `GET` | `/api/issues/<issue_id>/attachments` | List attachments on an issue. |

```bash
# Create
curl -X POST http://127.0.0.1:3100/api/companies/<id>/issues \
  -H "Content-Type: application/json" \
  -d '{"title":"…","description":"…","status":"todo","priority":"medium"}'

# Transition
curl -X PATCH http://127.0.0.1:3100/api/issues/<id> \
  -H "Content-Type: application/json" \
  -d '{"status":"in_progress"}'
```

!!! note "origin_kind / origin_id are SQL-only"
    The cross-company dispatch link (`origin_kind`, `origin_id`) is not exposed through the REST surface — the SDK strips it. Plugins write it via direct SQL against `PAPERCLIP_DB` (port `54329`). See [craft-dispatch](../components/plugins/craft-dispatch.md) for the reference flow.

### Projects

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/companies/<company_id>/projects` | List projects in a company. Craft-dispatch uses the first project as the default destination. |

### Agents (company members)

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/companies/<company_id>/agents` | List agents/members. Returns `{id, name, role, title, icon, status, adapterType, adapterConfig, runtimeConfig, ...}`. |
| `GET` | `/api/companies/<company_id>/members` | Alias surface used by nexus-mcp; same shape. |

### Plugins

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/plugins` | List installed plugins with their full manifests (tools, hooks, version, apiVersion). |
| `POST` | `/api/plugins/install` | Install a plugin. Body required — empty body returns 500. |

### Routines

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/companies/<company_id>/routines` | List routines for a company. Each routine has `triggers` (cron schedules), `concurrencyPolicy`, `catchUpPolicy`. |
| `POST` | `/api/companies/<company_id>/routines/<routine_id>/trigger` | Manually fire a routine outside its schedule. |

### Labels

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/companies/<company_id>/labels` | List labels (company-scoped). |

## Error model

Standard HTTP codes:

| Code | Meaning |
|---|---|
| `200` | OK — body is the JSON result. |
| `400` | Request shape rejected. `GET /api/issues` returns `400` because it requires a company scope. |
| `404` | Resource (or route) not found. Several historical paths — `/api/sessions`, `/api/config`, `/api/agents`, `/api/metrics/scheduler-flow` — return `404` on the current instance; they live in older docs but are not part of the live surface. |
| `500` | Server error. Returned by `POST /api/plugins/install` when the body is missing. |

Errors come back as `{"error": "API route not found"}` or similar plain-JSON bodies.

## Conventions

- **Reads are non-locking.** Anyone can query state at any time.
- **Writes are durable before responding.** A `200` means the row has hit the embedded Postgres WAL.
- **State transitions are atomic.** `PATCH /api/issues/<id>` with `{status: …}` is one operation; if the transition is illegal the request is rejected before the row changes.
- **No authentication.** The server binds to `127.0.0.1` and the deployment mode is `local_trusted`. Anything that can reach `:3100` can drive the API.

## Configuration that affects the API

Plugin and CLI clients read these to find the API. Defaults work for the local install.

| Env var | Default | Used by |
|---|---|---|
| `PAPERCLIP_API_BASE` | `http://127.0.0.1:3100/api` | Every plugin and `cockpit/src/lib/paperclip.ts`. |
| `PAPERCLIPAI_HOST` | `127.0.0.1` | The server itself (bind address). |
| `PAPERCLIPAI_PORT` | `3100` | The server itself (bind port). |
| `PAPERCLIP_DB` | `postgresql://paperclip:paperclip@127.0.0.1:54329/paperclip` | Plugins that bypass the REST API for `origin_kind` / `origin_id` writes. |

See [Environment Variables](env-vars.md) for the full list.

## Documented partially

The following are referenced in source but not fully documented here because the surface area or schema was not reachable from a live probe at the time of writing:

- `POST /api/plugins/install` — accepted (`200` on `OPTIONS`), but the body schema isn't documented inline in the plugin code; see the [plugins overview](../components/plugins/index.md) for the install flow.
- Webhook subscription endpoints (referenced in the ACP plugin) — emitted via the Paperclip event bus, not via REST.

## See also

- [Paperclip component](../components/paperclip.md) — what this API is part of
- [Nexus MCP](../components/nexus-mcp.md) — the chairman-level MCP that bundles many of these calls into compound tools
- [Heartbeat concept](../concepts/heartbeat.md) — consumes the issue endpoints to pick up backlog
- [Tickets concept](../concepts/tickets.md) — the state machine the issue endpoints enforce
- [Environment Variables](env-vars.md) — `PAPERCLIP_API_BASE`, `PAPERCLIP_DB`, port settings
