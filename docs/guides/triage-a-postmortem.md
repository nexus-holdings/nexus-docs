# Triage a Postmortem

<p class="lede">This guide walks you through what to do when the postmortem pipeline has filed an ADR: read the record, action any eval gap or catalog change it identified, close the follow-up ticket, and confirm the regression is caught next time.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-20</span>
  <span>Owner: Platform</span>
</div>

!!! info "Prerequisites"
    - You know the postmortem ADR number (e.g. `ADR-031`) — usually surfaced via a Paperclip ticket labelled `postmortem`
    - `$NEXUS_ROOT` points at `~/Projects/nexus/`
    - You have push access to `eval-registry/` and the relevant catalog repo (`agent-catalog/` or `routine-catalog/`)
    - You've skim-read the [Postmortems concept page](../concepts/postmortems.md)

## 1. Read the postmortem record

Postmortems are numbered ADRs in `docs/decisions/`. Title prefix is `ADR-NN: Post-Mortem`.

```bash
# what this is — list every postmortem ADR currently filed
ls "$NEXUS_ROOT/docs/decisions/" | grep -iE "postmortem|post-mortem"

# verification: at minimum you should see 031- and 035- (the two pre-existing examples)
```

Open the one you're triaging:

```bash
# example: read ADR-035 (TEL adapter misconfiguration postmortem)
${PAGER:-less} "$NEXUS_ROOT/docs/decisions/035-tel-adapter-misconfig-postmortem.md"
```

The four sections to read first:

| Section | What it tells you |
|---|---|
| **What Failed** | Concrete failure mode — drawn from ticket title + error signal |
| **Root Cause** | The underlying mechanism — drawn from handoff report |
| **Eval Gap Identified** | Optional. If non-empty, an eval that would have caught this didn't exist |
| **Catalog Change Triggered** | Optional. If non-empty, an agent prompt / routine schedule / similar needs to change |

If both optional sections are empty, the postmortem is informational only — file it, no follow-up work. If either is populated, continue.

## 2. Read the linked Paperclip ticket

The pipeline files a follow-up ticket automatically (labelled `postmortem`). It's the work item that closes the loop.

```bash
# what this is — find tickets with the postmortem label
curl -s "http://127.0.0.1:3100/api/companies/$COMPANY_ID/issues" \
  | python3 -c "
import json,sys
for i in json.load(sys.stdin):
    if 'postmortem' in [l.get('name','') for l in i.get('labels',[])]:
        print(i['identifier'], i['status'], '-', i['title'])
"

# verification: at least one ticket per filed postmortem ADR; status typically "todo" or "in_progress"
```

The ticket body mirrors the four ADR sections via the `POSTMORTEM_ISSUE_TEMPLATE` in `nexus-core/nexus/execution/postmortem.py`.

## 3. If `eval_gap_identified` is set: design the new eval

The keyword-match step in [DFA 20](../concepts/postmortems.md#the-pipeline-dfa-20) names a missing eval type. Your job is to propose a concrete eval component that would have caught the failure.

```bash
cd "$NEXUS_ROOT/eval-registry"
git checkout -b add-eval-<eval-type>-<descriptor>

# inspect existing specs in the same eval type
ls specs/
ls specs/code-quality/   # or whatever type the gap named
```

Draft a new YAML spec under `specs/<eval-type>/<next-version>.yaml`. Reference an existing version in that directory for the schema — the format is documented in [Eval Registry](../components/eval-registry.md#the-spec-format).

Minimum additions for a new eval component:

| Field | Why |
|---|---|
| `id` + `version` | The version increments on every additive change |
| `target.kind` + `target.filter` | What the eval applies to (PR, session, design doc) |
| `scoring.type` | `weighted-rubric` / `llm-graded` / `deterministic` |
| `scoring.components` | The actual checks — each one with a weight |

Open the PR:

```bash
git add specs/<eval-type>/<new-version>.yaml
git commit -m "feat(evals): add <eval-type>/<descriptor>

Addresses gap identified by ADR-NN postmortem on ticket <short-id>.
Caught-by-this-eval check: <describe the rule>."
git push -u origin add-eval-<eval-type>-<descriptor>
gh pr create --fill

# verification: gh pr view returns the PR URL; CI runs eval-spec validator
```

## 4. If `catalog_change_triggered` is set: prepare the catalog PR

The triggered change is usually in one of two catalogs:

| Catalog | Typical change |
|---|---|
| `agent-catalog/` | Prompt patch (`vN.M.x+1`), skill list edit, model_config update |
| `routine-catalog/` | Schedule edit, concurrency policy change, issue_template fix |

Worked example for an agent prompt patch:

```bash
cd "$NEXUS_ROOT/agent-catalog"
git checkout -b fix-<agent-id>-<short-rationale>

# what this is — bump the version (patch since prompt-only) and edit base_prompt
$EDITOR agents/<agent-id>.yaml

# verification: the version bumped, the changelog gained a new entry citing the ADR
diff <(git show HEAD:agents/<agent-id>.yaml) agents/<agent-id>.yaml | head -40
```

The changelog entry MUST cite the postmortem ADR:

```yaml
changelog:
  - version: "2.3.1"
    change: "Tighten scope discipline on multi-file edits"
    justification: "ADR-035 — agent silently drifted into TUI mode when prompt left adapter choice ambiguous"
```

Push and open the PR — same convention as [Create an Agent](create-an-agent.md).

## 5. Close the follow-up ticket once changes are merged

After the eval PR + catalog PR are merged:

```bash
# what this is — comment the resolution, then move ticket to done
curl -X POST "http://127.0.0.1:3100/api/issues/$POSTMORTEM_TICKET_ID/comments" \
  -H "Content-Type: application/json" \
  -d '{"body":"Resolved.\n- eval-registry PR #<n> merged (adds <eval-id>)\n- agent-catalog PR #<n> merged (bumps <agent-id> to <version>)"}'

curl -X PATCH "http://127.0.0.1:3100/api/issues/$POSTMORTEM_TICKET_ID" \
  -H "Content-Type: application/json" \
  -d '{"status":"done"}'

# verification: GET the ticket; status should now be "done"
curl -s "http://127.0.0.1:3100/api/issues/$POSTMORTEM_TICKET_ID" | python3 -m json.tool | grep status
```

Note: per the [tickets state machine](../concepts/tickets.md#what-done-guarantees), `done` for a code-change ticket also requires the feature branch to be on `main`. The catalogs are PR-merged, so this is automatically satisfied — but the rule still applies.

## 6. Verify the regression is caught next time

Two checks, both empirical:

**A. Replay the original failing ticket against the patched agent / routine.** Refile it with the same title + description, watch dispatch:

```bash
curl -X POST "http://127.0.0.1:3100/api/companies/$COMPANY_ID/issues" \
  -H "Content-Type: application/json" \
  -d '{"title":"Regression test: <orig title>","description":"<orig body>","status":"todo","priority":"low","labels":["regression-replay"]}' \
  | python3 -m json.tool

journalctl --user -u nexus-heartbeat -f
```

Expectation: the ticket completes successfully, OR it fails with a *different* signal that the new eval would have surfaced earlier.

**B. Confirm the new eval ran.** After the replay finishes, check the metrics DB:

```bash
psql -d nexus_metrics -tAc \
  "SELECT eval_id, eval_version, score, dimensions
   FROM eval_results
   WHERE ticket_id = '<regression-ticket-id>'
   ORDER BY created_at DESC LIMIT 5;"

# verification: a row for the new eval type exists
```

## 7. Update the ADR status

Once the loop is closed (regression caught, follow-up ticket done), bump the ADR's status from `Accepted` (the default for postmortems) to `Closed` and add a brief resolution paragraph at the bottom. Send it via a normal docs PR.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Postmortem ADR exists but no follow-up ticket | Issue-filing step of the pipeline errored (non-fatal — see `postmortem.paperclip_issue_id == None`) | File the ticket manually using the `POSTMORTEM_ISSUE_TEMPLATE` shape |
| `eval_gap_identified` populated but no obvious eval-component fit | Keyword match was too coarse | Add a sharper keyword or propose a new eval type entirely (PR the `eval-registry` README first) |
| Catalog change PR rejected — "no KPI evidence" | New catalog entries require evidence in the changelog `justification` | Cite the postmortem ADR explicitly; CI's lint rule accepts ADR refs as evidence |
| Regression replay still fails the same way | The catalog patch didn't actually address the root cause | Re-read the ADR's "Root Cause" section; the fix may need a different prompt phrasing or a new tool |
| Two postmortems point at the same root cause | The gap-scan ran twice on related failures before either fix landed | Resolve both with the same eval + catalog PR; cross-link the ADRs |

## See also

- [Postmortems](../concepts/postmortems.md) — the concept page, including the DFA 20 pipeline diagram
- [Decisions Index](../concepts/decisions-index.md) — all ADRs including the two existing postmortems
- [Eval Registry](../components/eval-registry.md) — spec format + the five existing eval types
- [Agent Catalog](../components/agent-catalog.md) — versioning rules for prompt bumps
- [Debug a Ticket](debug-a-ticket.md) — the upstream guide when triaging the originating failure
- [Nexus Core](../components/nexus-core.md) — where `postmortem.py` lives
