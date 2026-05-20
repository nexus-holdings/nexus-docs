# Create a Routine

<p class="lede">This guide walks you through adding a new routine to the catalog: pick the trigger and concurrency policy, draft the YAML, PR it, register it with Paperclip, and verify it fires.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-20</span>
  <span>Owner: Platform</span>
</div>

!!! info "Prerequisites"
    - `~/Projects/nexus/routine-catalog/` is checked out and pushable
    - Paperclip is running on `127.0.0.1:3100`
    - You know the owning company slug (e.g. `nexus-engineering`) and its UUID — `export COMPANY_ID=<uuid>`
    - For cron routines, you've decided the schedule + timezone
    - For webhook routines, you know what external system will POST to Paperclip

## 1. Decide the trigger type

| Trigger | When to use | Example existing routine |
|---|---|---|
| `cron` | Fixed cadence — health checks, sweeps, daily reports | `health-check` (every 15 min) |
| `webhook` | Reactive — fires when an external event arrives | (used by `company-heartbeat` for fast paths) |
| `api` | Manual-only — fired by an operator or another routine | Rare; reserved for orchestration helpers |

The walked example below adds a daily `secret-scan-sweep` cron routine for `nexus-engineering`.

## 2. Decide the concurrency policy

Three options, named for what happens when a fire arrives while a previous run is still active:

| Policy | Behavior | Use when |
|---|---|---|
| `skip_if_active` | Drop the new fire; let the existing run finish | Most routines — idempotent, double-running wastes work |
| `coalesce_if_active` | Merge the new fire into the current run's queue | Cumulative routines — missing a fire matters |
| `always_enqueue` | Always queue; serialise as they arrive | High-stakes routines where every fire must be recorded |

Default to `skip_if_active` unless you have a documented reason to choose differently.

## 3. Draft the YAML

```bash
cd "$NEXUS_ROOT/routine-catalog"
git checkout -b add-secret-scan-sweep

# what this is — start from the closest existing peer
cp routines/stale-ticket-sweep.yaml routines/secret-scan-sweep.yaml
$EDITOR routines/secret-scan-sweep.yaml

# verification: yaml parses and id matches filename
python3 -c "import yaml; d=yaml.safe_load(open('routines/secret-scan-sweep.yaml')); assert d['id']=='secret-scan-sweep'"
```

Reference YAML:

```yaml
id: secret-scan-sweep
version: "1.0.0"
company: "nexus-engineering"
category: maintenance
title: "Daily secret-scan sweep"
description: |
  Run secret-scan across every active company repo once a day.
  Posts a ticket with findings if anything new shows up.

trigger:
  type: cron
  schedule: "0 7 * * *"        # daily at 07:00 local
  timezone: "Europe/Amsterdam"

concurrency_policy: skip_if_active
catch_up_policy: skip_missed

variables: []

issue_template:
  title: "Daily secret-scan sweep"
  description: |
    Automated routine: sweep every active company repo with the secret-scan skill.

    Steps:
    1. List all companies with status=active
    2. For each, clone (or pull) the repo, run secret-scan
    3. Aggregate findings; group by company + severity
    4. Post a summary comment; flag any high-severity finding as a separate ticket
  priority: medium

paperclip_config:
  projectId: null
  assigneeAgentId: null

changelog:
  - version: "1.0.0"
    change: "Initial catalog entry"
    justification: "Catch leaked secrets before they ship; the manual cadence missed twice in Q2"
```

## 4. Validate the schema across the catalog

```bash
# what this is — parse every routine and assert required keys
for f in routines/*.yaml; do
    python3 -c "
import yaml; d=yaml.safe_load(open('$f'));
required=['id','version','company','category','title','description','trigger','concurrency_policy','catch_up_policy','issue_template','changelog']
missing=[k for k in required if k not in d]
assert not missing, f'$f missing {missing}'
print('OK', d['id'])
"
done

# verification: every line starts with "OK" — no AssertionError
```

## 5. PR the routine

```bash
git add routines/secret-scan-sweep.yaml
git commit -m "feat(routines): add secret-scan-sweep

Daily 07:00 sweep across every active company. Catches leaks the
manual cadence missed in Q2."

git push -u origin add-secret-scan-sweep
gh pr create --fill

# verification: gh pr view returns the PR URL
```

Reviewers check: the cron expression is valid 5-field, the timezone is IANA, `company` references a real company slug, and `concurrency_policy` is justified if it's not `skip_if_active`.

## 6. Register with Paperclip

The ACP plugin auto-provisions routines on startup. After the PR merges, restart Paperclip so the loader picks up the new file:

```bash
sudo systemctl restart paperclip
sleep 3

# what this is — confirm Paperclip's view of routines for the owning company
curl -s "http://127.0.0.1:3100/api/companies/$COMPANY_ID/routines" \
  | python3 -m json.tool \
  | grep -B1 -A4 secret-scan-sweep

# verification: a JSON object with the routine ID, triggers, concurrencyPolicy, catchUpPolicy
```

If the routine isn't there, check the Paperclip log:

```bash
journalctl -u paperclip -n 100 | grep -i "secret-scan-sweep\|routine"
```

## 7. Manually fire the routine

You don't want to wait until tomorrow at 07:00 to find out the YAML is wrong. Fire it once by hand:

```bash
# what this is — manual trigger via the Paperclip routines endpoint
ROUTINE_ID=$(curl -s "http://127.0.0.1:3100/api/companies/$COMPANY_ID/routines" \
  | python3 -c "import json,sys; r=[x for x in json.load(sys.stdin) if x['id'].endswith('secret-scan-sweep')]; print(r[0]['id'] if r else '')")

curl -X POST "http://127.0.0.1:3100/api/companies/$COMPANY_ID/routines/$ROUTINE_ID/trigger" \
  -H "Content-Type: application/json" \
  -d '{}' | python3 -m json.tool

# verification: response indicates the routine was triggered; an issue should appear
```

Alternative path: the `nexus-mcp` server exposes a `trigger_routine` compound tool — useful from inside an agent session.

## 8. Confirm an issue was spawned

```bash
# what this is — look for a fresh issue matching the routine's issue_template title
curl -s "http://127.0.0.1:3100/api/companies/$COMPANY_ID/issues" \
  | python3 -m json.tool \
  | grep -B1 -A3 "Daily secret-scan sweep"

# verification: an issue with status "todo" or "in_progress" and a recent createdAt
```

Once the heartbeat picks it up on the next tick, the routine's first real run is observable end-to-end.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Routine never appears in `/api/companies/<id>/routines` after PR merge | Paperclip didn't restart, or the YAML failed validation | `systemctl restart paperclip`; check `journalctl -u paperclip` for parser errors |
| Cron expression accepted but routine never fires on schedule | Wrong timezone, or cron spec is non-standard 6-field instead of 5-field | Use a 5-field expression; set `timezone` to a valid IANA name |
| Manual trigger returns 404 | `ROUTINE_ID` is wrong — the catalog ID and the Paperclip UUID differ | Fetch with the curl above; use the Paperclip-assigned UUID for the trigger call |
| Routine fires but no issue created | `issue_template` malformed, or `assigneeAgentId` references a missing agent | Set `assigneeAgentId: null`; ensure `priority` is one of `critical/high/medium/low` |
| Two routines fire at the same instant | Multiple routines schedule on the same minute | Stagger schedules (e.g. `0 7` and `5 7`); concurrency policies are per-routine, not global |

## See also

- [Routine Catalog](../components/routine-catalog.md) — component reference + the nine shipped routines
- [Paperclip API — Routines](../reference/api-paperclip.md#routines) — endpoint reference
- [Heartbeat](../concepts/heartbeat.md) — the per-company-heartbeat routine is the canonical example
- [Create a Company](create-a-company.md) — needed before a company-scoped routine can be registered
- [CLI Commands](../reference/cli-commands.md) — `nexus-mcp` `trigger_routine` and related tools
