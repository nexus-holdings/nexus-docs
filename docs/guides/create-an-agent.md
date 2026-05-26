# Create an Agent

<p class="lede">This guide walks you through adding a new agent to the catalog: pick the role, draft the YAML, send it through PR review, deploy into a company, and validate against a test ticket.</p>

<div class="page-meta">
  <span class="badge"><span class="dot"></span> living document</span>
  <span>Updated 2026-05-20</span>
  <span>Owner: Platform</span>
</div>

!!! info "Prerequisites"
    - `~/Projects/nexus/agent-catalog/` is checked out and pushable
    - At least one company exists in Paperclip ([Create a Company](create-a-company.md))
    - You have a target company UUID — `export COMPANY_ID=<uuid>`
    - The heartbeat timer is enabled — `systemctl --user is-active nexus-heartbeat.timer`

## 1. Pick a role + category

The catalog uses twelve categories. Choose the one that best fits the new role; this drives skill-matching during dispatch.

| Category | Examples already in catalog |
|---|---|
| `intake` | `intake-creative`, `intake-explicit`, `intake-implicit`, `intake-pragmatic` |
| `backend-dev` | `backend-engineer`, `backend-code-writer`, `backend-acceptance-validator` |
| `frontend-dev` | `frontend-engineer`, `frontend-component-builder`, `frontend-ui-designer` |
| `design` | `design-architect`, `design-ux-planner` |
| `devops` | `devops-engineer`, `devops-deployment-validator`, `devops-pipeline-writer` |
| `security` | (see catalog) |
| `review` | (see catalog) |
| `ticket-creation` | `pm-classifier`, `product-manager` |
| `improvement` | `improvement-writer`, `improvement-feedback-analyst`, `improvement-risk-assessor` |
| `research` | `autoresearch-evaluator`, `autoresearch-mutator` |
| `leadership` | `company-lead` |
| `management` | `qa-engineer`, `fullstack-engineer` (cross-cutting) |

The full list lives in `agent-catalog/agents/`. Read the closest existing peer before drafting — most new agents are minor variations on an existing one.

## 2. Draft the YAML

Create `agent-catalog/agents/<id>.yaml` following the schema. Worked example below — substitute the role you're adding.

```bash
cd "$NEXUS_ROOT/agent-catalog"
git checkout -b add-data-engineer

# what this is — start from an existing peer; backend-engineer is the cleanest reference
cp agents/backend-engineer.yaml agents/data-engineer.yaml
$EDITOR agents/data-engineer.yaml

# verification: yamllint catches indentation + structural issues
python3 -c "import yaml; print(yaml.safe_load(open('agents/data-engineer.yaml'))['id'])"
# → data-engineer
```

The minimum viable file:

```yaml
id: data-engineer
version: "1.0.0"
category: backend-dev

base_prompt: |
  You are a senior data engineer. You design and implement reliable ETL pipelines,
  data models, and analytics infrastructure using Python, SQL, dbt, and Airflow.

  Your responsibilities:
  - Validate schema assumptions before writing transforms
  - Write idempotent, re-runnable pipeline steps
  - Add data-quality assertions (not just unit tests)
  - Document table contracts in the wiki
  - Open a PR when done; never push directly to main

  You are stateless. All context comes from the wiki, the ticket, and this prompt.

skills:
  - python
  - sql
  - dbt
  - airflow
  - data-modelling

tools:
  - github-mcp
  - claude-code

model_config:
  default: sonnet
  complex_tasks: opus
  simple_tasks: haiku

performance:
  metrics: {}
  by_scenario: {}
  by_model: {}

changelog:
  - version: "1.0.0"
    change: "Initial catalog entry"
    justification: "Bootstrapping a data-engineering capability for Ledgerly"
```

## 3. Sanity-check the schema

```bash
# what this is — list every catalog YAML and confirm yours parses
for f in agents/*.yaml; do
    python3 -c "import yaml, sys; d=yaml.safe_load(open('$f')); assert d.get('id') and d.get('version') and d.get('base_prompt'), '$f missing required fields'" \
        || echo "BROKEN: $f"
done

# verification: no "BROKEN" lines printed
```

## 4. Set `model_config` defaults sensibly

The three keys map to dispatch decisions:

| Key | When used | Typical value |
|---|---|---|
| `default` | Normal dispatch path | `sonnet` for most roles, `opus` for safety-critical, `haiku` for high-throughput cheap work |
| `complex_tasks` | When the ticket has labels like `architecture`, `migration`, `security-sensitive` | `opus` |
| `simple_tasks` | When the ticket has `mechanical` label or matches cheap-route heuristics | `haiku` |

A `mechanical`-labelled ticket flows into `simple_tasks`; an `architecture`-labelled ticket flows into `complex_tasks`. Anything else lands on `default`.

## 5. Open the PR

The catalog is **PR-gated**. There is no `POST` endpoint for adding agents.

```bash
git add agents/data-engineer.yaml
git commit -m "feat(agents): add data-engineer

Initial catalog entry. KPI evidence will be populated by the metrics
pipeline once the agent has run against >= 10 tickets."

git push -u origin add-data-engineer
gh pr create --fill

# verification: gh pr view returns the PR URL; CI runs yaml-lint + schema-validator
```

Reviewers check three things: schema validity, prompt clarity (no contradictions, no leakage of company-specific details into a "universal" role), and that the changelog `justification` cites real evidence (or marks the entry as `Bootstrap`).

## 6. Deploy into a company

Once the PR is merged, the agent exists in the catalog but no company has it. Companies opt in via `nexus_deploy_cli.py agents`.

```bash
cd "$NEXUS_ROOT/nexus-core"

# what this is — preview which agents would be deployed
uv run python scripts/nexus_deploy_cli.py agents \
    ~/Projects/nexus-holdings/<company-slug> \
    --dry-run

# verification: output lists data-engineer among the agents to be staged
```

Then run for real (drop `--dry-run`).

## 7. Verify the agent is loaded by Paperclip

```bash
# what this is — confirm Paperclip sees the new agent in the company's roster
curl -s "http://127.0.0.1:3100/api/companies/$COMPANY_ID/agents" \
  | python3 -m json.tool \
  | grep -B1 -A3 data-engineer

# verification: a JSON object with the agent's name, role, adapterType, and adapterConfig
```

If the agent does not appear, the catalog loader picks it up on the next Paperclip start. Restart with `sudo systemctl restart paperclip` and re-check.

## 8. Dispatch a test ticket scoped to that agent

File a ticket whose labels + acceptance criteria match the new agent's skills:

```bash
curl -X POST "http://127.0.0.1:3100/api/companies/$COMPANY_ID/issues" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Validate data-engineer dispatch",
    "description": "Create a stub dbt project at dbt/ with one trivial model.\n\n## Acceptance criteria\n- dbt/dbt_project.yml exists\n- One .sql model under dbt/models/\n- dbt parse succeeds",
    "status": "todo",
    "priority": "low",
    "labels": ["data-engineering", "mechanical"]
  }' | python3 -m json.tool

# verification: ticket created with status "todo"
```

Watch the heartbeat pick it up:

```bash
journalctl --user -u nexus-heartbeat -f \
  | grep -E "dispatching|data-engineer"

# verification: line like "dispatching <ID> ... agent=data-engineer" within 5 minutes
```

When the session completes, the [Eval Registry](../components/eval-registry.md) records a `performance` entry against `data-engineer@1.0.0` — that's the first datapoint feeding future `model_config` tuning.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `nexus_deploy_cli.py agents` complains about missing skill tag | Catalog YAML references a skill not in the global skill list | Add to `agent-catalog/skills/` (or fix the typo); re-run |
| Paperclip 200s the agent list but the new agent is absent | Catalog loader cached the previous catalog at boot | `sudo systemctl restart paperclip` |
| Heartbeat dispatches the wrong agent for your test ticket | Skill match scores another agent higher | Tighten the new agent's `skills` to be more specific, or add a routing label to the ticket |
| YAML parses but agent never matches any ticket | `category` not in the canonical 12-list, or skill names aren't lowercase-kebab | Cross-check against `agents/backend-engineer.yaml` |
| KPI metrics never populate for the new agent | No completed sessions yet (need ≥10 for stable signal) | Run more tickets; check `nexus_metrics.performance_records` directly |

## See also

- [Agent Catalog](../components/agent-catalog.md) — component reference + full schema
- [Eval Registry](../components/eval-registry.md) — where agent performance lands after dispatch
- [Heartbeat](../concepts/heartbeat.md) — dispatch decision logic
- [Create a Company](create-a-company.md) — needs to exist before you can deploy an agent into it
- [CLI Commands](../reference/cli-commands.md) — `nexus_deploy_cli.py` invocation reference
