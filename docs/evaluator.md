# Evaluator

The Evaluator answers a single question for every Task or wave: **did this actually do
what it claimed to do?** The Contract Validator already checks that the output has the
right *shape*. The Evaluator checks that the output has the right *meaning*.

---

## 1. Why a separate component

Schema-valid does not mean correct. Examples of bugs the Evaluator catches:

- A "Monday.com task created" result whose `item_id` exists — but in the wrong board.
- A Google Sheet extraction that yields a perfectly formatted sheet — using the wrong source file.
- A grounded Slack response that quotes the right shape of citation — pointing at the wrong record.

Each of these passes the contract. None of them is what the user asked for.

---

## 2. What the Evaluator produces

For each evaluation pass it emits:

| Field | Description |
|-------|-------------|
| `evaluation_id` | Unique ID. |
| `task_id`, `wave_index` | Subject. |
| `criteria_version` | The versioned criteria suite run. |
| `criterion_results[]` | Per-criterion result: `pass`, `fail`, `unknown`, plus a justification. |
| `metrics[]` | Numeric metrics (similarity scores, confidence, latency, cost). |
| `verdict` | Aggregate `pass`, `fail`, `degraded`. |
| `proposed_criteria[]` | New criteria the Evaluator suggests adding, each with rationale. Always written as `candidate` review records, asynchronously — never pauses the Task. |
| `model_tier` | The cascade tier used (usually **L**). |
| `provenance` | Inputs, the routine version, the release manifest, and the data sampled. |

A `degraded` verdict means the output is acceptable but the Evaluator wants attention; it does not block.

---

## 3. Criteria

A **criterion** is the unit of evaluation:

| Field | Description |
|-------|-------------|
| `criterion_id`, `version` | Identity. |
| `intent` | What this criterion is checking, in plain English. |
| `applies_to` | Routine ID(s), policy ID(s), or `release_manifest_id` scope. |
| `kind` | `assertion`, `metric`, `judge` (LLM-based). |
| `spec` | The deterministic predicate, the metric formula, or the judge prompt. |
| `status` | `candidate`, `accepted`, `deprecated`, `archived`. (`candidate` replaces the older spelling `proposed` for consistency with the routine status enum — see [versioning.md §1](versioning.md).) |
| `accepted_by` | Human ID, or `policy:<policy_id>` if auto-accepted. |
| `provenance` | Where it came from: human-authored, Evaluator-proposed, derived from incident. |

Criteria are stored next to the routine or release they apply to, but they are
**first-class versioned objects** — they outlive any single Task.

---

## 4. Proposal-to-accept flow

The Evaluator's superpower is that it can spot a gap in its own coverage:

> "I noticed the task asserted `monday_item_id` exists, but did not assert it lives on the board the user named. Propose new criterion."

A proposal is written **asynchronously** to the criteria store as a `candidate` record. **The Task itself does not pause.** Proposals surface to humans through the review channels in [operations.md §2](operations.md), exactly like proposed policies.

```mermaid
flowchart LR
    EV[Evaluator] -->|writes async| PROP["candidate criterion"]
    PROP -->|human reviews via Slack/API/MCP| HUM{accept?}
    PROP -->|policy with evaluator_options.auto_accept_criteria=true| AUTO[auto-accept]
    HUM -->|yes| ACT["accepted criterion @1"]
    HUM -->|no| ARCH[archived]
    AUTO --> ACT
    ACT -->|next change| ACT2["accepted criterion @2"]
```

Acceptance options:

1. **Human acceptance** via the surface the proposal arrived on (typically Slack, optionally API/MCP). The accepting actor, timestamp, and reason are recorded.
2. **Policy-driven auto-accept.** A governance policy may declare `evaluator_options.auto_accept_criteria: true` (legacy alias: `auto_accept_evaluator_criteria`) for a specific routine or scope. Typically used for high-volume low-risk routines where reviewer fatigue would otherwise dominate.

**Rule.** Evaluator-generated criteria are *proposed* (status `candidate`) until accepted by a human or by a policy authorized to auto-accept. On acceptance the criterion is **versioned** (`@1`, `@2`, …) and **reused** via the criteria store and the next release manifest that references the routine/policy in question — see [versioning.md](versioning.md).

A concrete proposed criterion is in [`examples/criterion.example.yaml`](../examples/criterion.example.yaml).

---

## 5. When the Evaluator runs

| Trigger | Purpose |
|---------|---------|
| After every wave | Catch wave-local errors before the next wave consumes them. |
| Before terminal success | Final pass against the full criteria suite. |
| On-demand from a policy | A governance policy may demand evaluation at additional checkpoints. |
| Replay | Run accepted criteria against historical Tasks to validate a new version. |

---

## 6. Relationship to the contract validator and egress guards

Three checkers run on every Task. They are **complementary**, not substitutes.

| Concern | Validator | Egress Guards | Evaluator |
|---------|-----------|---------------|-----------|
| What it checks | **Shape** of inputs/outputs. | **Verifiability at egress**: evidence pointers, freshness, source authority, tenancy, tier/policy, budget. | **Meaning**: did the output actually do what the user asked. |
| Mechanism | Deterministic schema check. | Eight deterministic guards. See [knowledge-layer.md §5](knowledge-layer.md). | LLM-graded against criteria suite; usually **L** tier. |
| Cost | Very low. | Low (deterministic). | High. |
| Failure semantics | Revert state. | Block egress; emit `egress_blocked`; may `require_hitl`. | Flag `fail` / `degraded`. |
| Boundary | Every transition. | The `pre_egress` trigger phase, before the Tool Gateway. | Once per wave + once at end. |

The Evaluator is **not** a substitute for the egress guards. An Evaluator `pass`
verdict does **not** permit bypassing the deterministic guards: a result can be
meaning-correct against the criteria suite and still cite stale or non-authoritative
evidence. Both checks must pass on their own terms.

Conversely, the egress guards are not a substitute for the Evaluator. The guards
verify that *claims are checkable*; they do not verify that *claims are right*.

---

## 7. What the Evaluator is **not**

- Not a policy. Criteria do not block tool access; policies do.
- Not a benchmark suite. It evaluates real production Tasks, not synthetic prompts.
- Not optional in V1. The dogfood scenarios depend on it to catch wrong-but-schema-valid outputs (see [v1-dogfood-scenarios.md](v1-dogfood-scenarios.md)).
