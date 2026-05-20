# Debug a Ticket

<p class="lede">This guide walks you through diagnosing a ticket that misbehaved — stuck in <code>in_progress</code>, silently terminated, rejected by review, or blocked. It's a diagnostic flow, not a recipe: start at step 1, then branch by symptom.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-20</span>
  <span>Owner: Platform</span>
</div>

!!! info "Prerequisites"
    - Paperclip is running on `127.0.0.1:3100`
    - `journalctl --user` access to the `nexus-heartbeat` and `paperclip` units
    - The ticket UUID in hand — `export TICKET_ID=<uuid>`
    - Familiarity with the [ticket state machine](../concepts/tickets.md) (DFAs 1-4)

## 1. Read the current ticket state

Always start by pulling the canonical record from Paperclip — never trust the UI cache.

```bash
# what this is — the source-of-truth view of the ticket
curl -s "http://127.0.0.1:3100/api/issues/$TICKET_ID" | python3 -m json.tool

# verification: the response includes status, assignee, updatedAt, completedAt
```

Note the `status`, `updatedAt`, and `assignee`. The combination tells you which branch of the flow below to follow.

| Observed status | Likely symptom | Jump to |
|---|---|---|
| `in_progress` with old `updatedAt` | Stuck | Step 5 (stuck) |
| `in_progress` with recent `updatedAt` | Still legitimately running | Step 2 (transcript) |
| `in_review` with `updatedAt` >24h | Stuck in review | Step 6 (rejected/escalated) |
| `todo` after dispatch attempt | Heartbeat keeps skipping | Step 4 (heartbeat log) |
| `done` but branch not on main | Merge drift | Use `find_unmerged_done` MCP tool |
| `blocked` | Hit an unresolvable obstacle | Step 7 (blocked) |

## 2. Find the in-flight session (if any)

```bash
# what this is — list active sessions; correlate by ticket identifier
curl -s "http://127.0.0.1:3100/api/companies/$COMPANY_ID/agents" \
  | python3 -m json.tool

# Session transcripts on disk:
ls -lt ~/.claude/projects/*/  2>/dev/null | head -10
```

Then tail the matching JSONL transcript:

```bash
# what this is — live view of every agent turn, tool call, and tool result
tail -f ~/.claude/projects/-home-ian-Projects-<project>/<session-uuid>.jsonl \
  | python3 -m json.tool

# verification: you see assistant turns, tool calls, and tool results in real time
```

For a session that has already exited, the full transcript is in the same JSONL — read it from the top.

## 3. Read the comment trail

The agent posts structured comments on key transitions (start, blocker, completion). Pull them:

```bash
curl -s "http://127.0.0.1:3100/api/issues/$TICKET_ID/comments" \
  | python3 -m json.tool

# verification: comments appear chronologically; look for "[DONE]", "[BLOCKED]", reviewer verdicts
```

A healthy completed ticket has at least: a dispatch comment, a `[DONE]` comment from the implementer, and a verdict comment from the reviewer.

## 4. Check the heartbeat skip/dispatch log

The heartbeat decides every 5 minutes whether to dispatch a `todo` ticket. If yours keeps not getting picked up, the heartbeat is explicit about why.

```bash
# what this is — recent heartbeat decisions
journalctl --user -u nexus-heartbeat -n 200 \
  | grep -E "$(echo $TICKET_ID | cut -c1-8)|dispatching|skipping|memory infra|session cap"

# verification: lines explain dispatch decisions ("dispatching", "skipping", "memory infra unhealthy")
```

Common skip reasons:

| Log line | Meaning |
|---|---|
| `0/N sessions in use — at cap` | Session cap reached for this company |
| `memory infra unhealthy` | MemPalace API or Context-1 down — see [installation troubleshooting](../getting-started/installation.md#troubleshooting) |
| `no eligible agent for skill set X` | No agent in the company's roster matches the ticket's labels |
| `ticket already running` | A dispatch is already in-flight |

## 5. Symptom: stuck in `in_progress`

Use the `find_stuck_work` MCP tool (it scans all companies):

```bash
# what this is — call the chairman-level MCP tool from a shell-friendly Python entrypoint
uv run --project "$NEXUS_ROOT/nexus-mcp" python -c "
from nexus_mcp.server import find_stuck_work
print(find_stuck_work(hours_threshold=4))
"

# verification: JSON output lists every issue with status in_progress/in_review older than 4h
```

If your ticket is in the list, the implementer is silent. Inspect the session log on disk (step 2) — common patterns:

- Final entry is a tool call with no tool result → the tool stalled (network, subprocess hang)
- Frozen TUI banner with no further activity → adapter misconfiguration ([ADR-035](../concepts/decisions-index.md) is the canonical case)
- Last assistant turn says "I'll wait for X" → the agent is genuinely blocked but never posted `[BLOCKED]`

Recovery: PATCH the ticket back to `todo` for re-dispatch:

```bash
curl -X PATCH "http://127.0.0.1:3100/api/issues/$TICKET_ID" \
  -H "Content-Type: application/json" \
  -d '{"status":"todo"}'
```

## 6. Symptom: failed, silent, or rejected

For tickets that reached `in_review` and then got rejected, find the verdict:

```bash
curl -s "http://127.0.0.1:3100/api/issues/$TICKET_ID/comments" \
  | python3 -c "
import json,sys; cs=json.load(sys.stdin)
for c in cs:
    body=c.get('body','')
    if any(k in body.lower() for k in ['approve','reject','escalate','verdict']):
        print(c['createdAt'], '\n', body[:500], '\n---')
"

# verification: the verdict comment prints with timestamp + body
```

For sessions that silent-terminated (exit 0, no verdict), the postmortem pipeline fires automatically. See step 8.

## 7. Symptom: `blocked`

The agent has explicitly given up. Look at the most recent agent comment for the blocker rationale:

```bash
curl -s "http://127.0.0.1:3100/api/issues/$TICKET_ID/comments" \
  | python3 -c "
import json,sys; cs=json.load(sys.stdin); cs.sort(key=lambda c: c.get('createdAt',''));
for c in cs[-3:]:
    print(c.get('createdAt'), c.get('body','')[:400], '\n---')
"
```

Resolve the blocker (provide context, fix infra, decompose the ticket) then PATCH to `todo`.

## 8. If a postmortem was filed, read it

The postmortem pipeline runs automatically on terminal failures. Check the decisions tree:

```bash
# what this is — search for a postmortem ADR referencing this ticket UUID
grep -l "$TICKET_ID" $NEXUS_ROOT/docs/decisions/*.md 2>/dev/null

# verification: zero or more file paths; if any match, read them — they have the root-cause analysis
```

Then jump to [Triage a Postmortem](triage-a-postmortem.md) for follow-up.

## 9. Check the metrics record (sessions that completed)

For tickets that did finish — successfully or not — there's a row in `nexus_metrics.performance_records`:

```bash
# what this is — pull the agent's performance row for this ticket
psql -d nexus_metrics -tAc \
  "SELECT agent_id, model, outcome, duration_seconds, tokens_in, tokens_out
   FROM performance_records
   WHERE ticket_id = '$TICKET_ID'
   ORDER BY created_at DESC LIMIT 5;"

# verification: at least one row; outcome ∈ {success, failure, timeout, silent_term}
```

Cross-reference the outcome against the verdict comment from step 6.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `curl /api/issues/$TICKET_ID` returns 404 | Wrong UUID, or the ticket lives in a different Paperclip instance | Re-fetch via `/api/companies/<id>/issues` and grep for the short ID |
| Session JSONL not on disk | Session ran on a different host, or got cleaned up by `session-cleanup` routine | Check `journalctl -u paperclip` for the session ID; consult MemPalace once the promoter has mirrored |
| Heartbeat log has no entry for the ticket | Heartbeat hasn't fired since the ticket was created | Fire it: `systemctl --user start nexus-heartbeat.service` |
| `find_stuck_work` returns empty but the ticket is stuck visually | The `updatedAt` was bumped recently (e.g. a comment was posted) | Lower `hours_threshold`, or inspect the session log directly |
| Comments endpoint returns 200 with `[]` | The agent never started, or it crashed before posting | Step 4 — the heartbeat log will explain |

## See also

- [Tickets](../concepts/tickets.md) — the full lifecycle state machine
- [Heartbeat](../concepts/heartbeat.md) — the dispatch loop
- [Sessions vs. Tickets](../concepts/sessions-vs-tickets.md) — why these are different units
- [Postmortems](../concepts/postmortems.md) — what happens to terminal failures
- [Triage a Postmortem](triage-a-postmortem.md) — next step when a PM has been filed
- [Paperclip API — Issues](../reference/api-paperclip.md#issues-tickets) — endpoint reference
