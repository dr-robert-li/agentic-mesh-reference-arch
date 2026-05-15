# Governance

Governance is the layer that decides whether a Task — or a step inside one — is **allowed**
to proceed, and under what conditions. It is durable, versioned, and authored by humans
in a form they can read.

---

## 1. What a policy is

A **policy** is a persisted object with:

| Field | Description |
|-------|-------------|
| `policy_id` | Stable identifier. |
| `version` | Monotonic version. Each accepted change creates a new version. |
| `tenant_scope` | A tenant ID, a tenant group, or `global`. |
| `title` | Human-readable title. |
| `intent` | Natural-language description of what the policy is protecting. |
| `enforcement_mode` | `deterministic`, `agentic`, or `hybrid`. |
| `rule` | The deterministic rule (if applicable): a structured predicate over Task fields, tool calls, or evaluator results. |
| `agent_prompt` | The prompt and criteria given to an LLM judge (if agentic or hybrid). |
| `triggers` | When the policy fires: `on_task_create`, `on_plan`, `pre_tool_call`, `pre_egress`, `post_evaluation`, etc. |
| `on_violation` | What to do: `deny`, `require_hitl`, `downgrade_tier`, `route_to_routine`, `log_only`. |
| `provenance` | Authored-by (human or evaluator-proposed), source document, prior version. |
| `created_at`, `accepted_at`, `deprecated_at` | ISO timestamps. |
| `status` | `proposed`, `active`, `deprecated`, `archived`. |

A concrete shape is in [`examples/policy.example.yaml`](../examples/policy.example.yaml).

---

## 2. How policies are authored

Policies can enter the system three ways:

1. **API.** Direct structured upload (YAML/JSON) — typical for platform engineers.
2. **Slack.** A natural-language message in a designated channel: *"Never send Stripe data to BigQuery without a written request from finance."* The mesh compiles this into a policy proposal (status: `proposed`). A human reviewer accepts and it becomes `active`.
3. **Uploaded document.** A PDF / Markdown / Google Doc of the company's SOP. The mesh extracts policy candidates from the document, each as a `proposed` policy. A human reviewer accepts them individually.

The compilation step in (2) and (3) is itself a Task — it has provenance, is logged, and uses a large-tier model because policy authoring is high-stakes and ambiguous.

### 2.1 Policy trigger phases

Where a policy fires over the Task lifecycle:

```mermaid
flowchart LR
    A[on_task_create] --> B[on_plan]
    B --> C[pre_tool_call]
    C --> D[post_tool_call]
    D --> E[pre_egress]
    E --> F[post_evaluation]
```

A single policy can declare multiple `triggers`. `pre_tool_call` and `pre_egress` are the
two most common gates that fire `require_hitl`; `post_evaluation` is where policies
that depend on evaluator verdicts (or that auto-accept evaluator criteria) attach.

---

## 3. Deterministic vs agentic enforcement

| Mode | What runs | When to use |
|------|-----------|-------------|
| **Deterministic** | A schema-checkable predicate using **set membership** over the trust-tier enum (e.g. `tool.tier in {"write_revocable", "write_destructive"} AND tenant.region == "eu"`) | When the rule is exact, hot-path, and fast. Most "you can't call X without Y" rules. |
| **Agentic** | An LLM judge invoked with policy text and Task context | When the rule depends on intent or natural-language nuance (e.g. "we shouldn't book external dependencies in customer-facing copy"). |
| **Hybrid** | Deterministic prefilter, agentic only on near-misses | Default for ambiguous policies with a hot path. |

Agentic enforcement uses the **L** model tier. Deterministic uses no LLM. Hybrid uses **S/M** for prefilter, **L** only on escalation.

**Trust tiers are an unordered enum, not an ordinal scale.** Policies must use set
membership (`in`, `not in`) rather than comparison operators. Writing `tool.tier >
read_safe` is undefined behavior. See [tool-plans.md §1](tool-plans.md) for the tier
list.

---

## 4. Lifecycle

```mermaid
stateDiagram-v2
    [*] --> proposed
    proposed --> active : human accepts<br/>(or auto-accept policy)
    proposed --> archived : rejected
    active --> deprecated : superseded
    deprecated --> archived
    active --> active : new version (n+1)
```

- Policies are **never deleted**. `archived` is the floor.
- A new version of an active policy creates a new row; the previous version is `deprecated` immediately but still resolvable by past Tasks via their `release_manifest_id`.

---

## 5. Persistence guarantees

- Policies persist **indefinitely**. There is no TTL.
- A Task's `release_manifest_id` pins the exact policy versions in effect at creation time. A policy change mid-flight does **not** silently apply.
- A long-running Task may opt in to live-policy updates only via the side-by-side versioning flow in [orchestration.md](orchestration.md).

---

## 6. HITL flow

When `on_violation = require_hitl`:

1. Orchestrator transitions the Task to `AWAITING_HITL`, recording `hitl.from_state` (the state the gate fired from — typically `POLICY_CHECK` or `EVALUATING`).
2. A notification is sent to the policy's configured channel (Slack channel, API webhook, MCP subscriber).
3. The notification carries: actor, entrypoint, timestamp, policy ID + version, reason, and a **recommended action**.
4. An approver responds via the same channel; the response becomes a `hitl_decision` provenance entry.
5. On **approve**: Orchestrator resumes the Task to `hitl.from_state`. On **deny**: Orchestrator transitions the Task to `DENIED`.

A concrete decision record is in [`examples/hitl-decision.example.json`](../examples/hitl-decision.example.json).

```mermaid
sequenceDiagram
    participant O as Orchestrator
    participant G as Policy Engine
    participant S as Slack
    participant H as Human
    participant L as Logs
    O->>G: precheck(task)
    G-->>O: violation: require_hitl
    O->>L: decision: hitl_requested
    O->>S: post approval prompt
    H->>S: approve / deny / approve with edits
    S->>O: hitl_decision
    O->>L: decision: hitl_resolved
    O->>O: resume or DENY
```

A worked example is in [v1-dogfood-scenarios.md §5](v1-dogfood-scenarios.md).

---

## 7. Relationship to the Evaluator

The Evaluator may *propose* new criteria. A criterion is not a policy — it lives next to a routine or release.

**Rule.** Evaluator-generated criteria are written as `candidate` review records and remain
proposals until either (a) a human accepts them, or (b) a policy with
`auto_accept_evaluator_criteria: true` authorizes auto-acceptance for the criterion's
scope. On acceptance, the criterion is versioned (`v1`) and reused via release manifests.
A Task is **not paused** by criterion proposals — see [task-contract.md §2.1](task-contract.md).

A policy that authorizes auto-acceptance carries an `evaluator_options` block:

```yaml
evaluator_options:
  auto_accept_criteria: true        # canonical name; alias of auto_accept_evaluator_criteria
  scope:
    routine_refs: [rt_tldv_to_monday]
    severity_at_most: degraded      # only auto-accept criteria whose worst observed verdict is `degraded`
```

The legacy spelling `auto_accept_evaluator_criteria` remains a recognized alias. Use
`evaluator_options.auto_accept_criteria` in new examples for consistency.

See [evaluator.md](evaluator.md) and [`examples/policy.example.yaml`](../examples/policy.example.yaml).

---

## 8. What governance is **not**

- It is not a workflow engine. Policies do not decide *what* to do; they decide *whether*.
- It is not a rate limiter. Quotas live in the Tool Gateway.
- It is not the validator. Schema correctness lives in the Contract Validator.
- It is not the evaluator. Output quality lives in the Evaluator.
