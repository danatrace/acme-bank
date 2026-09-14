# dt-workflow-building — repo context supplement

This file adds repo-specific context on top of the base skill. The full skill definition (pre-flight checklist, schema reference, subworkflow catalog, connector catalog) lives in [`.agents/skills/dt-workflow-building/SKILL.md`](../.agents/skills/dt-workflow-building/SKILL.md).

---

## Naming conventions in chat (mandatory)

When narrating workflow behaviour back to the user in chat, always refer to **Davis** as **"Dynatrace Intelligence"** — this includes "Davis problem", "Davis-detected", "Davis event", "Davis-problem trigger", etc. Use phrasing like *"a Dynatrace Intelligence-triggered workflow"* or *"fires on a Dynatrace Intelligence problem"*.

This applies **only to user-facing prose**. Inside `.workflow.json` files keep Dynatrace API values verbatim: `event.kind == "DAVIS_PROBLEM"`, `triggerConfiguration.type: "davis-problem"`, `dt.davis.*` fields, existing workflow titles, etc. Never rewrite those tokens — the platform requires them exactly as-is.

---

## Repo-specific remediation registry

Some scenarios in this repo already have a canonical, deployed remediation workflow. Do **not** generate a new one for them. Always: (a) check the tenant for the named workflow, (b) if missing, `dtctl apply -f` the file from this repo, (c) trigger it or let its Davis problem trigger fire automatically.

| Scenario | Canonical file | Workflow title in tenant |
|---|---|---|
| Postgres optimizer regression / CPU throttling on `acme-bank/postgres` | [`workflows/workflow-acme-remediate-postgres-cpu-on-problem.workflow.json`](../workflows/workflow-acme-remediate-postgres-cpu-on-problem.workflow.json) | `Acme remediate postgres CPU (pod restart on Davis problem)` |
| accounts DB connection pool leak / Response time Degradation | [`workflows/workflow-acme-remediate-accounts-pool-leak-on-problem.workflow.json`](../workflows/workflow-acme-remediate-accounts-pool-leak-on-problem.workflow.json) | `Acme remediate accounts DB pool leak (rolling restart on Davis problem)` |
| web-ui login progressive degradation — SLOWDOWN → ERROR on `web-ui` | [`workflows/workflow-acme-remediate-webui-login-on-problem.workflow.json`](../workflows/workflow-acme-remediate-webui-login-on-problem.workflow.json) | `Acme remediate web-ui login degradation (rolling restart on Davis problem)` |
| Ollama CPU saturation / throttling on `ollama` deployment | [`workflows/workflow-acme-remediate-ollama-cpu-on-problem.workflow.json`](../workflows/workflow-acme-remediate-ollama-cpu-on-problem.workflow.json) | `Acme remediate ollama CPU saturation (raise CPU limits on Davis problem)` |
| loadgen pods stuck in Pending / RESOURCE_CONTENTION on `acme-bank` | [`workflows/workflow-acme-remediate-loadgen-pods-pending-on-problem.workflow.json`](../workflows/workflow-acme-remediate-loadgen-pods-pending-on-problem.workflow.json) | `Acme remediate loadgen pods stuck in pending (recreate job on Davis problem)` |
| Ordered pod restart by CPU — preventive / scheduled | [`workflows/workflow-acme-restart-acme-bank-pods-by-cpu.workflow.json`](../workflows/workflow-acme-restart-acme-bank-pods-by-cpu.workflow.json) | `Acme restart acme-bank pods ordered by CPU (scheduled, every 3h)` |
| Bug-toggle ConfigMap activator — shared enable/disable subworkflow | [`workflows/subworkflow-acme-bug-toggle-via-ssm.workflow.json`](../workflows/subworkflow-acme-bug-toggle-via-ssm.workflow.json) | `Acme bug-toggle ConfigMap activator (SSM)` |

---

## Connector input typing gotcha — array fields must be real JSON arrays

The Dynatrace AWS connector (and most others) validates task `input` fields at **execution** time, not at import time. `dtctl apply` will happily import a workflow whose `Filters` (or any other array-typed field) is encoded as a JSON string — but the first execution fails with:

```
Error: Unable to process the given payload, the following fields are violating the specification:
   - Filters: Invalid input: expected array, received string
```

Opening + saving the workflow in the UI silently re-normalizes the field, which is why "save in the UI and it works" masks the bug.

**Rule:** in `.workflow.json` files, array-typed connector inputs must be real JSON arrays. For `dynatrace.aws.connector:ec2-describe-tags`:

```jsonc
// correct
"Filters": [
  { "key": "resource-type", "value": "instance" },
  { "key": "value", "value": "{{ input()['host_name_tag'] }}" },
  { "key": "key", "value": "Name" }
]

// wrong — passes import, fails first execution
"Filters": "[{\"key\":\"resource-type\",\"value\":\"instance\"}, ...]"
```

The same rule applies to any connector input typed as `array`. When unsure, run `dtctl get workflow <id> -o json` on a known-good workflow in the tenant and copy its structural shape.
