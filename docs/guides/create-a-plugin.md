# Create a Plugin

<p class="lede">This guide walks you through building a minimal Paperclip plugin: scaffold from the SDK, declare one tool and one event subscription, build it, install it against the running Paperclip, and verify the tool reaches an agent.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-20</span>
  <span>Owner: Platform</span>
</div>

!!! info "Prerequisites"
    - Node 22+ and `npm` (managed via `mise` per the global stack)
    - Paperclip is running on `127.0.0.1:3100`
    - `$NEXUS_ROOT` points at `~/Projects/nexus/`
    - You have a sudoers entry for `systemctl restart paperclip.service` (the scoped one used by the ACP plugin)
    - You've skim-read [Plugins Overview](../components/plugins/index.md)

## 1. Scaffold from an existing minimal plugin

The smallest first-party plugin is `paperclip-plugin-memory` (~five tools, one event log line). Clone its layout as the scaffold:

```bash
cd "$NEXUS_ROOT"

# what this is — copy memory plugin layout as the starting skeleton
cp -R paperclip-plugin-memory paperclip-plugin-hello
cd paperclip-plugin-hello

# strip the memory-specific source so we can re-write src/ from scratch
rm -rf dist node_modules src/memory-api.ts src/types.ts src/constants.ts

# verification: package.json + tsconfig.json + src/ + an empty dist target remain
ls
```

Edit `package.json` to rename the package:

```json
{
  "name": "paperclip-plugin-hello",
  "version": "0.1.0",
  "type": "module",
  "paperclipPlugin": {
    "manifest": "./dist/manifest.js",
    "worker": "./dist/worker.js"
  }
}
```

## 2. Write `src/manifest.ts`

Declare ID, one capability, one tool, one event subscription.

```ts
// src/manifest.ts
import type { PaperclipPluginManifestV1 } from "@paperclipai/plugin-sdk";

const manifest: PaperclipPluginManifestV1 = {
  id: "paperclip-plugin-hello",
  apiVersion: 1,
  version: "0.1.0",
  displayName: "Hello",
  description: "Minimal example plugin — one tool, one event subscription.",
  author: "you",
  categories: ["automation"],
  capabilities: [
    "events.subscribe",
    "agent.tools.register",
    "activity.log.write",
  ],
  entrypoints: { worker: "./dist/worker.js" },
  tools: [
    {
      name: "hello_say",
      displayName: "Say hello",
      description: "Return a greeting for the given name.",
      parametersSchema: {
        type: "object",
        properties: { name: { type: "string" } },
        required: ["name"],
      },
    },
  ],
};

export default manifest;
```

```bash
# verification: TypeScript accepts the manifest shape
npx tsc --noEmit src/manifest.ts
```

## 3. Write `src/worker.ts`

Implement the tool handler and the event subscriber. Per the SDK README, `runWorker(plugin, import.meta.url)` is the line that keeps the worker alive when launched by the host.

```ts
// src/worker.ts
import { definePlugin, runWorker, type ToolResult } from "@paperclipai/plugin-sdk";
import manifest from "./manifest.js";

const plugin = definePlugin({
  async setup(ctx) {
    ctx.logger.info("hello plugin: setup");

    ctx.events.on("issue.created", async (event) => {
      ctx.logger.info("hello plugin: saw issue.created", {
        issueId: event.entityId,
      });
    });

    ctx.tools.register(
      "hello_say",
      {
        displayName: "Say hello",
        description: "Return a greeting for the given name.",
        parametersSchema: manifest.tools![0].parametersSchema,
      },
      async (params): Promise<ToolResult> => {
        const { name } = params as { name: string };
        return { content: `Hello, ${name}!` };
      },
    );

    ctx.logger.info("hello plugin: 1 tool registered");
  },
});

export default plugin;
runWorker(plugin, import.meta.url);
```

## 4. Build

```bash
# what this is — TypeScript compile to dist/
npm install
npm run build

# verification: dist/worker.js + dist/manifest.js exist
ls dist/
```

If `npm run build` is not defined yet, add it to `package.json` (`"build": "tsc"`).

## 5. Install against the running Paperclip

For local development, install by `packagePath`:

```bash
# what this is — point Paperclip at the local plugin directory
curl -X POST http://127.0.0.1:3100/api/plugins/install \
  -H "Content-Type: application/json" \
  -d '{"packagePath":"'"$NEXUS_ROOT"'/paperclip-plugin-hello"}' \
  | python3 -m json.tool

# verification: response contains "id": "paperclip-plugin-hello" and "status": "installed" (or similar)
```

If the install path errors with `500`, the request body is probably missing or malformed — the install endpoint returns 500 on empty bodies (see [Paperclip API](../reference/api-paperclip.md#plugins)).

## 6. Confirm Paperclip loaded the manifest

```bash
# what this is — list installed plugins with their full manifests
curl -s http://127.0.0.1:3100/api/plugins \
  | python3 -m json.tool \
  | grep -B1 -A6 "paperclip-plugin-hello"

# verification: the listing includes "tools": [{"name":"hello_say",...}]
```

## 7. Verify the tool reaches an agent

The tool will show up in an agent session's tool list at session start. Dispatch a ticket scoped to a company that has at least one configured implementer:

```bash
curl -X POST "http://127.0.0.1:3100/api/companies/$COMPANY_ID/issues" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Plugin smoke test: call hello_say",
    "description": "Call the hello_say tool with name=\"world\" and post the result as a comment.\n\n## Acceptance criteria\n- A comment containing the string \"Hello, world!\" is posted on this ticket",
    "status": "todo",
    "priority": "low",
    "labels": ["mechanical"]
  }' | python3 -m json.tool

# verification: ticket created with status "todo"
```

Then tail the heartbeat + ACP logs while the agent runs:

```bash
journalctl --user -u nexus-heartbeat -f &
journalctl -u paperclip -f | grep -E "hello_say|hello plugin"
```

You should see the worker log line "hello plugin: 1 tool registered" at install time, and "hello plugin: saw issue.created" when the new ticket lands.

## 8. Trigger the subscribed event

Filing the ticket from step 7 already emits `issue.created`. If you want to fire it again without creating new tickets, do a no-op patch:

```bash
# what this is — emit issue.updated (which the subscriber doesn't listen to)
# OR simply file another ticket. There is no public "emit arbitrary event" endpoint.
curl -X PATCH "http://127.0.0.1:3100/api/issues/<ticket_uuid>" \
  -H "Content-Type: application/json" \
  -d '{"priority":"medium"}'

# verification: paperclip log shows the event handler firing for issue.updated only if you also subscribed to it
```

For event-bus debugging, the plugin's `ctx.logger` lines are the canonical signal — visible via `journalctl -u paperclip`.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `npm install` fails resolving `@paperclipai/plugin-sdk` | The SDK isn't published yet; only available inside existing plugin checkouts | Copy `node_modules/@paperclipai` from a sibling plugin (e.g. `paperclip-plugin-memory`) |
| `POST /api/plugins/install` returns 500 | Body shape wrong, or `packagePath` doesn't exist on disk | Re-check the path; ensure JSON is well-formed; consult `journalctl -u paperclip -n 50` |
| Plugin listed but tools don't appear in agent sessions | Missing `agent.tools.register` capability in the manifest | Add the capability, rebuild, reinstall |
| Worker crashes silently after `setup` | `runWorker(plugin, import.meta.url)` not called as the last line | Restore the call; without it the worker exits immediately |
| Subscribed event handler never fires | Wrong event name (e.g. `issue_created` instead of `issue.created`) | Cross-reference the core domain events list in the plugin-SDK README |

## See also

- [Plugins Overview](../components/plugins/index.md) — capabilities, event bus, lifecycle
- [Memory plugin](../components/plugins/memory.md) — the minimal-but-real reference plugin
- [Paperclip API — Plugins](../reference/api-paperclip.md#plugins) — install + list endpoints
- [ACP plugin](../components/plugins/acp.md) — the canonical big-plugin example (agent lifecycle, sessions)
- [Decisions Index](../concepts/decisions-index.md) — DFA 23 (plugin lifecycle) in `state-machines.md`
