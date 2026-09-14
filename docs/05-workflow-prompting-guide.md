# How to prompt the agent to build a Dynatrace workflow

This guide helps you get precise, hallucination-free results when asking the AI agent to build or edit Dynatrace `.workflow.json` files in this repo.

The agent uses the `dt-workflow-building` skill internally, which prevents most schema mistakes. But the agent still relies on **what you put in the prompt**. The more facts you supply up front, the fewer round-trips and the fewer mistakes.

---

## Required context to include in your prompt

1. **Workflow purpose in one sentence** — what should fire, on what condition, doing what action.
   - Good: "Restart the postgres statefulset when Davis fires the high-CPU-throttling problem on the <cluster-name> cluster."
   - Bad: "Build a postgres workflow."
2. **Trigger type** — pick one explicitly:
   - **Manually triggered** (admin tool, no event): say "manual / `trigger: {}` / inputs only".
   - **Problem-triggered**: say "Problem trigger" AND give a **sample problem ID (P-…)** whose **title (`event.name`) and cluster (`k8s.cluster.name`)** the workflow should match. The agent fetches the problem via the Dynatrace MCP and uses those values in the filter so the workflow fires on **any future problem with the same title on the same cluster** — not just that one P-ID. Do NOT ask the agent to guess names from descriptions.
   - **Event/log/metric-triggered**: provide the filter DQL or a reference workflow UUID to clone.
3. **Reference workflow UUID** (strongly recommended) — "clone the trigger/structure from `<uuid>`". This bypasses schema guessing entirely. The agent will run `dtctl get workflow <uuid>` and use its shape as ground truth.
4. **Target / scope facts**:
   - Cluster name, namespace, deployment/statefulset/job name (exact strings)
   - EC2 Name tag of the host that runs the SSM agent
   - AWS region
   - AWS connection token name or "use the existing one from `<file>`"
5. **HITL or autonomous** — if remediation, say which. Default to HITL if unsure.
6. **File scope** — "new file at `workflows/<name>.workflow.json`" OR "edit existing `workflows/<name>.workflow.json`". Don't say "the postgres workflow" if two exist.
7. **Should it auto-fire?** — say `isActive: true` or `isActive: false` explicitly. The agent has shipped workflows in both states unintentionally before.
8. **Import after?** — "also import via dtctl" vs "just write the file". Default: write only.

---

## Anti-hallucination guardrails

The agent will follow these automatically (they live in the skill), but knowing them helps you spot mistakes:

- The agent **must** consult the `dt-workflow-building` skill before writing any `.workflow.json`. If the agent jumps straight to JSON without printing the pre-flight checklist, push back.
- For Problem triggers the agent **must** either clone a real workflow via `dtctl get` or use the Problem trigger template in the skill — not write the trigger from memory.
- The agent **must not** trust the Dynatrace docs MCP for workflow schema questions — it hallucinates fields like `entityFilter.includeEntities`.
- The agent **must** test `filterQuery` via `execute-dql` before import and verify with `dtctl get workflow <id>` after import.
- The agent **must not** filter Problem triggers by `affected_entity_ids` / entity UUIDs unless you explicitly demand it — entity IDs are unstable; problem name + cluster tag is canonical.

---

## Common failure modes and how to avoid them

| What goes wrong | Why | Prompt fix |
|---|---|---|
| Trigger silently absent after import | Wrong schema; `dtctl apply` returns OK even when the body is dropped | Give a reference workflow UUID to clone |
| Workflow fires on wrong problems | Filter built from a problem **description** instead of `event.name` | Give the problem ID; the agent will fetch real values |
| `dtctl apply` creates a duplicate every run | File has no `id` field — `apply` is create-only without it | Say "delete the existing `<uuid>` first, then re-import" |
| Filter rejected with HTTP 400 | Used `contains()` / `in()` (Grail-only, not EventTrigger) | Say "use `matchesValue` / `matchesPhrase`" or just trust the skill |
| Empty result from `ec2-describe-tags` → SSM task fails | `MaxResults` not set | Say "set `MaxResults: 100` on `ec2-describe-tags`" or trust the skill |
| Subworkflow re-invented from scratch | Agent didn't check the Subworkflow Collection | Say "reuse subworkflow `<uuid>`" or "check Section 12 first" |

---

## Five example prompts (copy-paste templates)

### 1) New Problem-triggered remediation workflow (HITL, SSM-based)

> Build a new HITL remediation workflow at `workflows/workflow-acme-remediate-redis-breaker.workflow.json` that fires on **any Davis problem with the same title and cluster as sample problem P-<id>** (clone the Problem trigger shape from reference workflow `<workflow-uuid>`; extract `event.name` + `k8s.cluster.name` from P-<id> and use them in the filter). On approval, run SSM `AWS-RunShellScript` against the EC2 host tagged `Name=<ec2-host-tag>` in `<region>` to patch the `workarounds` ConfigMap in namespace `acme-bank` setting `redis.breaker=on`, then restart deployment `accounts`. Use the existing AWS connection token from `workflows/workflow-acme-bug-toggle-via-ssm.workflow.json`. Set `MaxResults: 100` on `ec2-describe-tags`. `isActive: false` initially. Write the file only — do not import yet.

### 2) Edit an existing workflow's retry/timeout

> In `workflows/workflow-acme-remediate-postgres-cpu-on-problem.workflow.json`, change `tasks.wait-for-finish.retry.count` to 60 and `tasks.wait-for-finish.timeout` to 90000. Re-import: delete the current `<uuid>` first, then `dtctl apply`. Verify with `dtctl get`.

### 3) Build a new reusable subworkflow

> Build a new subworkflow at `subworkflows/subworkflow-acme-restart-k8s-resource.workflow.json` that takes inputs `{instanceId, awsRegion, namespace, target, dynatraceawsconnection}` and runs `microk8s kubectl -n <namespace> rollout restart <target>` via SSM, then waits using subworkflow `<workflow-uuid>`. Follow the canonical subworkflow schema in Section 1 of the skill. Title prefixed with 🧩. Do not import.

### 4) Manually-triggered admin workflow (no event)

> Build a manually-triggered admin workflow at `workflows/workflow-acme-scale-deployment.workflow.json` (no trigger, `trigger: {}`) that takes inputs `{host_name_tag, awsregion, namespace, deployment, replicas}` and runs `microk8s kubectl scale deployment <deployment> -n <namespace> --replicas=<replicas>` via SSM on the EC2 host tagged by `host_name_tag`. Set `MaxResults: 100` on `ec2-describe-tags`. Import via dtctl.

### 5) Add a new task to an existing workflow

> In `workflows/workflow-acme-bug-toggle-via-ssm.workflow.json`, add a new task `record-bizevent` after `wait-for-finish` that uses `dynatrace.automations:run-javascript` to POST a BizEvent with `{eventType:"acme.bug-toggle.applied", bugId:"<input.bug_id>", action:"<input.action>", ssmCommandId:"<result.toggle-bug-configmap.Command.CommandId>"}` to the OTLP endpoint. Predecessor: `wait-for-finish`. Re-import: delete `<current-uuid>` first, then apply, then verify with `dtctl get`.

---

## When the agent should push back

If your prompt is missing critical context, the agent should ask before writing JSON — specifically about: problem ID (for Problem triggers), exact target resource name, HITL vs autonomous, and whether to import. If the agent guesses any of these silently, push back and ask for the pre-flight checklist.
