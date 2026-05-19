# V1 Dogfood Scenarios

V1 is proven by making the mesh work on the team that builds it. The scenarios below are
the acceptance criteria for v0.1 of the implementing platform.

There are **three workflows** and **five cross-cutting scenarios** (the fifth was
added in v0.1.3 to exercise the deterministic egress guards on stale evidence). Each
cross-cutting scenario must be demonstrable on top of at least one of the workflows.

---

## 1. Workflow A — Create a Monday.com task from a tl;dv recording

**Entrypoint:** Slack slash command in an internal channel.
**Goal:** "Capture the action items from this tl;dv recording as a Monday.com task on board X."

### Inputs
- A tl;dv recording URL (or recording ID).
- The target Monday.com board (resolvable by name or ID).
- The Slack user, channel, and thread.

### Plan
- Wave 1 (parallel)
  - `tldv.fetch_transcript` — `read_sensitive`.
  - `monday.boards.resolve` — `read_safe`.
- Wave 2
  - Agent: extract action items from transcript (M-tier).
- Wave 3
  - `monday.items.create` — `write_revocable`. Idempotency key = `task_id:wave3:step1`.
- Wave 4
  - `slack.chat.postMessage` — `write_revocable`. Returns the Monday.com item URL to the originating thread.

### Evaluator criteria (illustrative)
- The created Monday.com item exists on the *named* board, not just any board.
- The item's description includes at least one action item from the transcript.
- The Slack confirmation references the same item URL the executor reported.

### Governance touchpoints
- A policy may demand that tl;dv recordings tagged "external customer" require HITL before any write to Monday.com.

---

## 2. Workflow B — Extract a Slack-provided artifact into a Google Sheet

**Entrypoint:** Slack message action ("Use this with the mesh") on a message that contains a file or link.
**Goal:** "Extract the structured content of this artifact into a new Google Sheet and reply with the link."

### Inputs
- The Slack message (file attachment or URL).
- The target Drive folder (per-tenant default, optionally overridable).

### Plan
- Wave 1
  - Classify the artifact type (S-tier): CSV, PDF, table-in-doc, screenshot, JSON, etc.
- Wave 2 (parallel)
  - `drive.files.fetch` — `read_sensitive`.
  - `agent.extract_table` (M-tier) — produce a normalized table.
- Wave 3
  - `sheets.create_with_data` — `write_revocable`. Idempotency key as above.
- Wave 4
  - `slack.reply_in_thread` — `write_revocable`. Returns the *correct* sheet link.

### Evaluator criteria
- The sheet link in the Slack reply matches the sheet the executor reports having created (catch wrong-but-schema-valid).
- The sheet's row count is plausible against the artifact size.
- Header row is present and non-empty.

### Egress guards
- Before wave 3 (`sheets.create_with_data`), the Orchestrator runs the eight
  deterministic egress guards (see [knowledge-layer.md §5](knowledge-layer.md)) on
  the proposed write. Guards 2 (claim-evidence map), 4 (freshness), and 5
  (source authority) verify that the source artifact pointed at by the Slack
  message is the one being written, and that its evidence pointer is fresh.
- Before wave 4 (`slack.reply_in_thread`), the guards re-run on the sheet link
  itself — guard 5 confirms `drive.files` is authoritative for `sheet.id`.

### Why this scenario matters
This is the canonical "wrong but schema-valid" failure mode: the executor reports
*a* link; the evaluator must confirm it's *the* link — and the deterministic egress
guards must confirm the link points at evidence the mesh actually fetched.

---

## 3. Workflow C — Review a Monday.com task and emit a grounded Slack response

**Entrypoint:** Slack slash command referencing a Monday.com item.
**Goal:** "Summarize the state of this task and tell me what's blocking it, with citations."

### Inputs
- Monday.com item ID or URL.
- The Slack user, channel, thread.

### Plan
- Wave 1
  - `monday.items.fetch_with_history` — `read_sensitive`.
- Wave 2
  - Agent: build a grounded response (L-tier for synthesis), citing specific updates and field changes.
- Wave 3
  - `slack.chat.postMessage` — `write_revocable`. Posts the response in-thread.

### Evaluator criteria
- Every claim in the response references a real `update_id` or field change on the item.
- The "blocked by" diagnosis matches at least one update marked as a blocker or matches the most recent state-change reason.

### Egress guards
- The agent emits a `claim_evidence_map_ref` sidecar (see
  [`examples/claim-evidence-map.example.json`](../examples/claim-evidence-map.example.json))
  mapping every claim to a Knowledge Layer entry sourced from
  `monday.items.fetch_with_history`.
- Before wave 3 (`slack.chat.postMessage`), the Orchestrator runs the eight
  deterministic egress guards. Any stale evidence pointer fails guard 4 (freshness)
  and the egress is blocked — the response is not posted until the evidence is
  refetched.

### Why this scenario matters
Demonstrates the **grounded response** pattern — the answer is checkable against
its sources. The Evaluator enforces that the citations *match* the claims; the
deterministic egress guards enforce that the citations are *resolvable, fresh, and
sourced from systems authoritative for the predicate*.

---

## 4. Cross-cutting scenario 1 — Governance violation requiring HITL

**Setup.** A policy `P_external_customer_writes` is active:
> "Any write to Monday.com on a board tagged `external-customer` requires written approval from the project lead."

`enforcement_mode: agentic`. `on_violation: require_hitl`.

**Trigger.** Run Workflow A against a recording that resolves to an external-customer engagement.

**Expected behavior.**
1. After plan generation, the policy fires before wave 3.
2. Task transitions to `AWAITING_HITL`.
3. A Slack DM goes to the configured approver with: actor, entrypoint, timestamp, policy ID + version, the proposed Monday.com create payload, the recommended action.
4. Approver clicks **Approve**.
5. Task resumes; wave 3 executes; provenance now includes a `hitl_decision` entry.

**Negative path.** Approver clicks **Deny** with reason; Task transitions to `DENIED`; Slack thread is updated with the reason.

---

## 5. Cross-cutting scenario 2 — Drift / version-control on a long-running Task

**Setup.** A long-running Task created with `release_manifest_id = R1` is in progress
(say, a daily-cohort processing routine). An updated policy `P_v2` is published, producing
`release_manifest_id = R2`.

**Expected behavior.**
1. The Task does **not** auto-migrate.
2. The Orchestrator forks a side-by-side run under `R2`.
3. At reconciliation, outputs are compared. If they agree within tolerance, the Task adopts `R2` for subsequent waves. If they disagree, the Task transitions to `AWAITING_HITL` with a diff in the notification.
4. The Task's provenance now records both manifests.

---

## 6. Cross-cutting scenario 3 — Persistent routine lifecycle

**Setup.** Workflow A has run twice successfully. A routine `rt_tldv_to_monday@1` has been
written with status `candidate` based on those Tasks.

**Steps to demonstrate.**
1. **Recall.** A user runs Workflow A; the planner *recalls* the candidate routine instead of planning from scratch.
2. **Update.** A reviewer accepts the routine (status -> `active`); later they `update` it to write `rt_tldv_to_monday@2` (also `active`); v1 transitions to `deprecated`.
3. **Pin.** A pin binding is written for `scope: tenant:T -> rt_tldv_to_monday@2` so the routine is stable through subsequent updates.
4. **Copy.** Tenant T' copies the routine; provenance records the source.
5. **Deprecate / archive.** When `v3` lands and a new pin is written, `v2` transitions to `deprecated` and eventually `archived`; historical Tasks still resolve `v2`.

This scenario validates the full operation set in [versioning.md §4.1](versioning.md).

---

## 7. Cross-cutting scenario 4 — Duplicate garbage collection

**Setup.** A user fires Workflow A. Within 12 minutes, the same user fires what looks like the same request (same recording, same board, slightly different wording).

**Expected behavior.**
1. The second Task transitions to `AWAITING_DUP_CONFIRM`.
2. Slack message: "This looks like the request you made at 10:33. Are these the same? [Yes, same] [No, different]". Includes a link to the original Task.
3. User clicks **Yes, same**. Second Task transitions to `SUPPRESSED`. A 24-hour suppression window opens for that `(actor, fingerprint)`.
4. A third near-identical request within the 24-hour window is auto-suppressed (no prompt). The user is pointed at the original Task.
5. After 24 hours, the suppression expires; the next near-identical request prompts again.

The "false-match" path (user clicks **No, different**) is also exercised: the new Task continues, and a false-match hint is recorded so this specific near-miss pair does not re-prompt.

---

## 8. Cross-cutting scenario 5 — Stale-evidence egress block

**Setup.** Workflow C is run against a Monday.com item whose Knowledge Layer entry
was fetched 20 minutes ago. The tool plan declares `freshness_ttl_seconds: 900` for
`monday.items.fetch_with_history` (see [tool-plans.md §3.3](tool-plans.md)), so the
cached entry is `stale`.

**Trigger.** The agent produces a grounded Slack response that cites the stale
Knowledge Layer entry in its `claim_evidence_map_ref` sidecar.

**Expected behavior.**

1. Before wave 3 (`slack.chat.postMessage`), the Orchestrator runs the eight
   deterministic egress guards (see [knowledge-layer.md §5](knowledge-layer.md)).
2. Guard 4 (freshness) fires: the cited evidence is `stale`. The egress is blocked.
3. A `decision_kind: egress_blocked` Decision log entry is emitted with the failing
   guard named.
4. The Orchestrator either (a) refetches `monday.items.fetch_with_history` through
   the Tool Gateway, updates the Knowledge Layer entry, re-runs the guards and
   proceeds, or (b) if `evidence_fetch_budget.max_refetches` is exhausted,
   transitions to `AWAITING_HITL` with `hitl.from_state = EGRESS_CHECK`.

**Why this scenario matters.** It exercises the *deterministic* leg of
"LLM-first verification with deterministic guards": the LLM produced a perfectly
formatted, evaluator-passing response; only the egress guard caught that the
underlying evidence was stale.

---

## 9. Acceptance summary

A V1 implementation is accepted when all three workflows and all five cross-cutting
scenarios can be demonstrated end-to-end with the audit chain — event, decision
(including `egress_blocked` / `egress_passed`), action, validation, evaluation —
fully populated, and with at least one accepted evaluator-proposed criterion visible
in the criteria store.
