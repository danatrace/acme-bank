# dt-workflow-building

A Claude Code skill for building and editing Dynatrace Automations **workflow JSON files** (`*.workflow.json`) — Davis problem triggers, schedule triggers, SSM patterns, loop iteration, HITL approvals, and the full connector catalog.

---

## What's in this branch

| Path | Purpose |
|---|---|
| [`.agents/skills/dt-workflow-building/SKILL.md`](.agents/skills/dt-workflow-building/SKILL.md) | Skill definition — loaded by Claude Code via `skills-lock.json` |
| [`.claude/skills/dt-workflow-building/SKILL.md`](.claude/skills/dt-workflow-building/SKILL.md) | Mirror copy for the Claude Code harness |
| [`skills-lock.json`](skills-lock.json) | Skill registry (single entry: `dt-workflow-building`) |
| [`docs/05-workflow-prompting-guide.md`](docs/05-workflow-prompting-guide.md) | Human guide: how to write prompts that get precise results |
| [`docs/06-workflow-building-skill.md`](docs/06-workflow-building-skill.md) | Repo-context supplement: naming conventions, remediation registry, connector gotcha |
| [`workflows/`](workflows/) | Seven canonical workflow JSON files used as reference examples |

---

## Canonical workflow examples

The seven files in [`workflows/`](workflows/) are the reference implementations the skill cites. Each demonstrates one or more patterns:

| File | Patterns demonstrated |
|---|---|
| [`workflow-acme-remediate-postgres-cpu-on-problem.workflow.json`](workflows/workflow-acme-remediate-postgres-cpu-on-problem.workflow.json) | Davis problem trigger · SSM pod restart · subworkflow call chain |
| [`workflow-acme-remediate-accounts-pool-leak-on-problem.workflow.json`](workflows/workflow-acme-remediate-accounts-pool-leak-on-problem.workflow.json) | Davis problem trigger · rolling deployment restart via SSM |
| [`workflow-acme-remediate-webui-login-on-problem.workflow.json`](workflows/workflow-acme-remediate-webui-login-on-problem.workflow.json) | Davis problem trigger · SLOWDOWN→ERROR cascade · rolling restart |
| [`workflow-acme-remediate-ollama-cpu-on-problem.workflow.json`](workflows/workflow-acme-remediate-ollama-cpu-on-problem.workflow.json) | Davis problem trigger · `kubectl set resources` via SSM · CPU limit remediation |
| [`workflow-acme-remediate-loadgen-pods-pending-on-problem.workflow.json`](workflows/workflow-acme-remediate-loadgen-pods-pending-on-problem.workflow.json) | Davis problem trigger · dynamic workload detection · conditional Job recreation |
| [`workflow-acme-restart-acme-bank-pods-by-cpu.workflow.json`](workflows/workflow-acme-restart-acme-bank-pods-by-cpu.workflow.json) | Schedule trigger (inactive on import) · DQL ranking · `withItems` loop · `concurrency: 1` |
| [`subworkflow-acme-bug-toggle-via-ssm.workflow.json`](workflows/subworkflow-acme-bug-toggle-via-ssm.workflow.json) | Reusable subworkflow · SSM send-command + wait · connector token reference |

---

## Prerequisites

| Tool | Purpose | Install |
|---|---|---|
| **`dtctl`** | Import and manage workflow JSON files against the Dynatrace tenant | `winget install Dynatrace.dtctl` (Windows) |
| **Dynatrace MCP server** | Live DQL queries, problem lookups, tenant introspection from within Claude Code | Configure in `.vscode/mcp.json` |

Configure `dtctl` for the `<your-tenant-id>` tenant:

```bash
dtctl config set-context <your-tenant-id> --environment https://<your-tenant-id>.apps.dynatrace.com
dtctl config use-context <your-tenant-id>
```

MCP server entry (`.vscode/mcp.json`):

```json
{
  "servers": {
    "dynatrace-mcp": {
      "url": "https://<your-tenant-id>.apps.dynatrace.com/platform-reserved/mcp-gateway/v0.1/servers/dynatrace-mcp/mcp",
      "headers": { "Authorization": "Bearer <platform-token>" }
    }
  }
}
```

---

## Quick start

The skill activates automatically when you ask Claude Code to build or edit a `.workflow.json` file. It enforces a **mandatory pre-flight checklist** (13 items) before writing any JSON — print it, tick each item, then produce the file.

**Minimal prompt for a new problem-triggered remediation:**

```
Build a new HITL remediation workflow at workflows/workflow-<name>.workflow.json that fires on any
Davis problem with the same title and cluster as sample problem P-<id>. Clone the trigger shape from
reference workflow <uuid>. On approval, run SSM AWS-RunShellScript on the EC2 host tagged
Name=<host-tag> in <region> to <action>. Use the AWS connection token from
workflows/subworkflow-acme-bug-toggle-via-ssm.workflow.json. isActive: false. Write only.
```

See [`docs/05-workflow-prompting-guide.md`](docs/05-workflow-prompting-guide.md) for five copy-paste templates and a full failure-mode guide.

---

## What the skill enforces

When invoked, the skill requires the agent to:

1. Print the **pre-flight checklist** with ✅/❌ before producing any JSON
2. Check the **subworkflow catalog** ([danatrace/dynatrace-subworkflow-collection](https://github.com/danatrace/dynatrace-subworkflow-collection), 48 subworkflows) before writing new logic
3. Test `filterQuery` via `mcp_dynatrace-mcp_execute-dql` before embedding it in a trigger
4. Verify the import via `dtctl get workflow <id> -o json` after apply — exit code alone is not sufficient

**Skill coverage:**

- Davis problem trigger — canonical filter template, verified DQL allowlist, `matchesValue` vs `contains`
- Schedule trigger — interval and cron, `isActive: false` for inactive-on-import
- Loop / iteration — `withItems` at task level (not nested), `concurrency`, correct `_.varname` access
- SSM + wait — canonical subworkflow reuse, `MaxResults: 100` on `ec2-describe-tags`
- HITL vs autonomous — `user-task` only when explicitly requested
- Jinja2 escaping — single-escape `\"`, prefer single quotes inside `{{ }}` inside nested JSON strings
- OOB connector catalog — AWS EC2/SSM/S3/ASG/IAM/EventBridge, Kubernetes, Azure, Slack, Jira, ServiceNow, Teams, PagerDuty, Email, Ansible, GitHub, GitLab, Jenkins, Snowflake

---

## Key conventions

- Workflow files are always `.workflow.json` — never `.workflow.yaml`
- No `id` field in files you author — Dynatrace assigns the UUID on import
- No emoji in parent workflow titles — `🧩subworkflow` prefix for subworkflows only
- `schemaVersion: 4` for new files
- Array-typed connector inputs (`Filters`, etc.) must be real JSON arrays — a string-encoded array imports silently but fails at first execution with `"expected array, received string"`

Full reference: [`.agents/skills/dt-workflow-building/SKILL.md`](.agents/skills/dt-workflow-building/SKILL.md)
