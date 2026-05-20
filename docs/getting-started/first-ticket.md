# First Ticket

End-to-end walkthrough: file a ticket, watch the heartbeat pick it up, observe the agent run, see it land in `done`. This assumes you've completed [Installation](installation.md) and have at least one company.

## Get your company ID

```bash
curl -s http://127.0.0.1:3100/api/companies | python3 -m json.tool
```

Note the UUID for the company you'll target. Export it for convenience:

```bash
export COMPANY_ID=<uuid-from-above>
```

## File the ticket

A well-formed ticket has: title, description with acceptance criteria, type, priority.

```bash
curl -X POST http://127.0.0.1:3100/api/companies/$COMPANY_ID/issues \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Add hello-world endpoint to demo service",
    "description": "Add GET /hello that returns {\"message\": \"hello\"}.\n\n## Acceptance criteria\n- New route registered\n- Returns 200 + JSON body as above\n- Unit test added and passing",
    "type": "feature",
    "priority": "medium",
    "labels": ["mechanical"]
  }' | python3 -m json.tool
```

A few things in that payload worth noticing:

- **Acceptance criteria** is in the description, scoped behind a `## Acceptance criteria` header — agents look for this.
- **`type: feature`** — drives which agent gets dispatched (an implementer, in this case).
- **`labels: ["mechanical"]`** — the `mechanical` label is one of several that signal "this is straightforward; a cheaper model is fine." See [Agent Catalog](../components/agent-catalog.md) for the full label-to-model routing table.

The response includes the ticket's short ID (e.g., `DEMO-1`) and its current status — `backlog`.

## Promote to `todo` (or let the heartbeat do it)

If you want to skip the wait, manually promote:

```bash
curl -X PATCH http://127.0.0.1:3100/api/issues/<TICKET_UUID> \
  -H "Content-Type: application/json" \
  -d '{"status": "todo"}'
```

Otherwise just wait — the heartbeat sweeps backlogs every 5 minutes.

## Heartbeat dispatches

On the next heartbeat tick, you'll see (in `journalctl --user -u nexus-heartbeat`):

```
[heartbeat] INFO  <company-slug>: 1 todo ticket(s), 0/2 sessions in use
[heartbeat] INFO  <company-slug>: dispatching DEMO-1 (priority=medium, label=mechanical)
[heartbeat] INFO  spawned agent <agent-id> for ticket DEMO-1 in branch ticket/<shortId>
```

The ticket transitions: `todo → in_progress`. An implementer agent starts working on a feature branch named `ticket/<shortId>`.

## Watch the agent work

!!! note "Cockpit is being rewritten"
    The original standalone Cockpit UI (port 3000) is archived. A replacement is being built as a plugin inside the governance company. Until that lands, the canonical way to watch a session is to tail the JSONL transcript on disk:

```bash
tail -f ~/.claude/projects/-home-ian-Projects-<your-project>/<session-uuid>.jsonl
```

In the live transcript you can see:

- Every assistant turn, tool call, and tool result
- Which files the agent is editing
- Which commands it's running
- The agent's reasoning between tool calls

If the archived Cockpit checkout is still running locally, it can also surface this data — but treat it as legacy.

## Implementer hands off to reviewer

When the implementer finishes, it pushes the branch and posts a completion comment:

```
[DONE] Implementation complete on ticket/dem01abc.
Branch pushed: ticket/dem01abc @ 9f3c2a4
Tests: 23 passing
Acceptance criteria addressed:
- ✅ Route registered
- ✅ Returns 200 + correct JSON
- ✅ Unit test added (test_hello_endpoint.py)
```

The ticket transitions: `in_progress → in_review`. A reviewer agent picks it up.

## Reviewer verdict

The reviewer reads the diff, runs verification, and posts one of three verdicts:

| Verdict | What happens |
|---|---|
| **Approve** | Merge-agent runs `git merge` and pushes. Ticket → `done`. |
| **Reject** | Ticket → `todo`. Implementer retries with reviewer feedback. |
| **Escalate** | Ticket stays `in_review`. A human gets paged. |

For a clean mechanical change like our hello-world example, expect approve.

## Merge to main

The merge-agent fast-forwards the feature branch to `main`. The ticket lands in `done`.

```bash
curl -s http://127.0.0.1:3100/api/issues/<TICKET_UUID> | python3 -m json.tool
# "status": "done"
# "completedAt": "2026-..."
```

## What was captured

The session transcript is now in MemPalace:

```bash
curl -X POST http://127.0.0.1:8102/v1/search \
  -H "Content-Type: application/json" \
  -d '{"query": "hello-world endpoint", "wing": "claude-nexus-agents", "max_results": 3}' | python3 -m json.tool
```

You should see your ticket's session transcript in the results. The promoter has also mirrored it into Context-1, so semantic search across all past work will find it the next time anyone (human or agent) asks about hello-world endpoints.

## What you learned

1. **Filing a ticket** is a single POST to [Paperclip](../components/paperclip.md).
2. **Acceptance criteria** in the description is how the agent knows when it's done.
3. **Labels** drive model selection — `mechanical` is cheap, no-label is the default model.
4. **The heartbeat** drives dispatch on a 5-minute cadence (or webhook-fast for hot events).
5. **Two agents** touch every ticket — implementer + reviewer. Plus merge-agent on approve.
6. **`done` means merged.** Not "reviewed and waiting" — actually on main.
7. **Memory captures everything.** Future agents will find this ticket via search.

## Where to go next

- [Companies](../concepts/companies.md) — what's actually holding the backlog
- [Tickets](../concepts/tickets.md) — the full state machine
- [Heartbeat](../concepts/heartbeat.md) — how dispatch works
- [Paperclip](../components/paperclip.md) — the orchestrator you've been POSTing to
- [Debug a Ticket](../guides/debug-a-ticket.md) — when things don't go cleanly
