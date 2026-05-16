# Operations

This document covers what humans see, what gets logged, how the mesh is monitored without
a frontend in V1, and how cost/quality are managed via the model cascade.

---

## 1. The five log streams

| Log | What it captures | Mandatory fields |
|-----|------------------|------------------|
| **Event** | External or internal triggers entering the mesh | `tenant_id`, `task_id`, `actor`, `entrypoint`, `timestamp`, `payload_ref` |
| **Decision** | Choices made by orchestrator, planner, or agentic policy | `tenant_id`, `task_id`, `actor`, `decision_kind`, `reason`, `release_manifest_id`, `timestamp` |
| **Action** | Side-effecting tool calls through the Gateway | `tenant_id`, `task_id`, `tool_id`, `tier`, `credential_alias`, `result_ref`, `timestamp` |
| **Validation** | Contract Validator runs | `tenant_id`, `task_id`, `wave_index`, `verdict`, `failures[]`, `timestamp` |
| **Evaluation** | Evaluator outputs and criteria activity | `tenant_id`, `task_id`, `criteria_version`, `verdict`, `proposed_criteria[]`, `model_tier`, `timestamp` |

Common fields across all logs: `entry_id` (ULID), `prev_entry_id` (forms a per-Task chain), `recommended_action` where applicable.

One sample line from each stream is in [`examples/logs.example.json`](../examples/logs.example.json).

### 1.0 Log / provenance flow

```mermaid
flowchart LR
    EVT[Event log<br/>external triggers] --> ORCH[Orchestrator]
    ORCH --> DEC[Decision log]
    ORCH --> EXEC[Executor]
    EXEC --> GW[Tool Gateway]
    GW --> ACT[Action log]
    ORCH --> VAL[Validator]
    VAL --> VLOG[Validation log]
    ORCH --> EV[Evaluator]
    EV --> ELOG[Evaluation log]

    DEC --> PROV[Task.provenance<br/>append-only refs]
    ACT --> PROV
    VLOG --> PROV
    ELOG --> PROV
    EVT --> PROV
```

Every log entry that needs to be re-surfaced to a human is referenced from `Task.provenance` by entry id.

### 1.1 Provenance fields surfaced for every human-facing notice

When a Task surfaces something to a human — an approval request, a duplicate confirmation,
an evaluator-proposed criterion — the message **must** include:

- **Actor** (who initiated the underlying work)
- **Entrypoint** (`slack`, `api`, `mcp`, `internal`)
- **Timestamp** (ISO-8601 UTC, plus a localized form)
- **Reason** (free text)
- **Recommended action**
- **Release manifest ID** and a one-line description of what changed since the last manifest, if relevant

This is the substrate that makes V1 monitorable without a UI.

---

## 2. Human monitoring without a frontend

In V1 there is no web UI. Humans interact with the mesh through three surfaces:

| Surface | What it's good for |
|---------|--------------------|
| **Slack** | Approvals, duplicate confirmations, evaluator proposals on the channel where the work originated. |
| **REST API** | Programmatic queries: `GET /tasks?state=AWAITING_HITL&tenant=T`, `GET /policies/proposed`, `GET /evaluations?verdict=fail`. |
| **MCP** | An external agent (including another mesh) can query the same surfaces using the same scope rules. |

### 2.1 Canonical queries

The implementing platform should expose at minimum:

- "What Tasks are paused waiting on me?" → API + Slack home tab listing.
- "What policies are proposed for this tenant?" → API + Slack DM digest.
- "What evaluator criteria are awaiting acceptance?" → API + Slack DM digest.
- "Show me the full audit trail of Task X" → API endpoint returning an interleaved view of events, decisions, actions, validations, evaluations.
- "What changed since release manifest R_n-1?" → API endpoint comparing two manifests.

---

## 3. Model cascade in practice

The cascade is an operational lever. Use the cheapest model that suffices and **record which tier was used and why** on every agent invocation.

| Tier | Default use |
|------|-------------|
| **S** | Intent classification, field extraction, simple lookup, duplicate-similarity scoring, status string normalization. |
| **M** | Most workflow steps, summarization, response formatting, routine planning from a recalled template. |
| **L** | Planning under ambiguity, governance edge cases (agentic enforcement), evaluation, policy compilation from natural language or document, version comparison, incident root-cause analysis. |

Operational rules:

1. Start every routine in **M**. Promote a step to **L** only when criteria show degradation.
2. **L** is mandatory for policy compilation and for agentic policy enforcement. Do not cascade these down silently.
3. The Evaluator runs at **L** by default. Cascading the Evaluator down requires a policy permitting it for the routine.
4. Per-Task cost and latency by tier are logged so cascading decisions are auditable.

---

## 4. Kill switch and DLQ operations

### 4.1 Kill switch
- Trippable from API or Slack (with appropriate role).
- Scopes: platform-wide, per-tenant, per-routine, per-policy.
- Every trip and untrip is a Decision log entry with actor, scope, reason.

### 4.2 DLQ
- DLQ entries are Tasks in `FAILED` state with `state_reason: poison_pill` and a reference to the offending payload.
- They are queryable via API (`GET /dlq?tenant=T`) and surfaceable in Slack.
- "Replay" creates a new child Task; the DLQ entry is itself never mutated.

---

## 5. Incident response

When something goes wrong, the canonical path is:

1. Find the affected Task(s) via API/MCP query.
2. Pull the full provenance chain.
3. Identify the responsible artifact (policy version, routine version, prompt version, tool plan version, platform release).
4. Trip the appropriate kill switch scope.
5. Open the postmortem; **L**-tier root-cause analysis is itself a Task that uses the same audit trail as its input.
6. Land the fix as a new version of the offending artifact (or a new platform release). Old Tasks stay pinned to the manifest under which they ran.

---

## 6. Cost and quality monitoring

The minimum operational telemetry to track:

- Tasks per state per tenant per day.
- Time-in-state distributions (especially `AWAITING_HITL` and `AWAITING_DUP_CONFIRM`).
- Evaluator verdict rates per routine.
- Proposed-vs-accepted criteria rate.
- Tool call counts per tier.
- Model tier distribution per routine.
- Duplicate-suppression rate.
- Kill switch trips and DLQ depth.

**Swarm / autonomy signals** (added in v0.1.2):

- **Spawn count** per Task, per routine, per tenant, per day.
- **Recursion depth** distribution per Task (max, p50, p95).
- **Wave count** per Task; the share of Tasks that exhausted `max_waves`.
- **Budget exhaustion rate** broken down by which budget tripped first
  (`max_waves`, `max_child_agents_per_wave`, `max_wall_clock_minutes`,
  `max_tool_calls`, `max_model_calls`, `max_token_budget`).
- **HITL escalation rate** broken down by trigger (policy boundary, high risk,
  budget exhaustion, low confidence, disagreement, external write, recursion
  threshold, kill-switch). See [swarm-supervisor.md §6](swarm-supervisor.md).
- **Kill-switch events** per scope (platform, tenant, routine, entrypoint, swarm
  depth), including runaway-swarm cancellations.
- **Candidate growth rate** — new candidates created per day, by source
  (human, planner, evaluator, supervisor).
- **Archive growth rate** — items moved to `archived` per day; the share that came
  from "unreviewed past TTL" vs explicit rejection.

In V1 these are exposed as API endpoints returning JSON. Dashboards belong to a later milestone.

---

## 7. Garbage collection of candidates and archived artifacts

Automatic candidate creation (see [versioning.md §4.3](versioning.md)) means the
candidate space grows continuously. A background GC sweep keeps it bounded.

### 7.1 Policies

| Class | Policy |
|-------|--------|
| **Duplicate candidates** | Within `duplicate_candidate_ttl`, identify near-duplicate candidates (same `routine_id`, structurally similar `decomposition` / `output_schema`). Keep the most provenance-rich; archive the rest. |
| **Unreviewed candidates** | Candidates older than `candidate_review_ttl` with no acceptance event are moved `candidate -> archived` automatically. The owning channel is notified once. |
| **Rejected candidates** | A candidate rejected by a human or by an auto-accept policy is moved directly to `archived` with a `rejected_by` provenance entry. |
| **Superseded candidates** | When a newer candidate for the same routine reaches `active`, older `candidate` rows for that routine are archived. |
| **Archived artifacts** | After `archive_retention_ttl`, an archived artifact becomes eligible for `prune`. Pruning writes a tombstone; the payload may be cold-stored or hard-deleted as the retention policy permits. |

### 7.2 Archive-before-hard-delete

This is invariant: **no artifact transitions directly from `active` or `candidate` to
hard-deleted state.** The required path is `… -> archived -> pruned`. The GC sweep
enforces this by refusing to skip the `archived` step even when the artifact has been
unreviewed for far longer than the archive retention TTL.

A policy may set `archive_retention_ttl` very short (e.g. one day) — that is allowed —
but it cannot set it to `0` and it cannot bypass `archived`.

### 7.3 Monitoring the GC sweep itself

The GC sweep emits Decision-log entries (`decision_kind: gc_swept`) per run so an
operator can answer:

- How many duplicates were consolidated last night?
- How many candidates aged out of review?
- How many archived artifacts were pruned, and against which retention policy?

These show up in the same canonical operator queries as kill-switch trips and
DLQ depth.
