# Changelog

All notable changes to this reference architecture are documented here.
The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and the project
uses [Semantic Versioning](https://semver.org/) for the **documentation** itself
(the implementing platform will track its own version).

## [0.1.2] — 2026-05-16

Documentation patch introducing the contractual **Swarm Supervisor** and closing
gaps in dynamic spawning, autonomy budgets, recursion semantics, and the candidate
lifecycle. Reference architecture only — no runnable code. Preserves v0.1.1 shape and
state-machine semantics; tightens the contract around autonomy and candidate
management.

### Added
- `docs/swarm-supervisor.md`: new document defining the Swarm Supervisor as a
  contractual control component for dynamic spawning. Covers V1 wave-based default,
  recursion semantics (enabled by default, bounded by depth/budget/policy), the
  five-step spawn gate (policy / budget / risk / provenance / marginal utility), the
  spawn-ledger fields, HITL escalation triggers, kill-switch and DLQ behavior for
  runaway swarms and poison-pill children, and the explicit statement that Temporal
  (or any other durable-workflow engine) is an **implementation candidate** for the
  control loop and **not itself the agent SDK**.
- `docs/orchestration.md` §3.1: dynamic-spawning and recursion semantics that share
  the existing wave machinery; recursion is allowed only inside policy / budget /
  risk boundaries.
- `docs/orchestration.md` §3.2: default conservative autonomy budget
  (`max_waves: 3`, `max_child_agents_per_wave: 5`, `recursion_enabled: true`,
  `max_recursion_depth: 1` with rationale, `max_total_spawns: 50`, with the explicit
  note that wall-clock / tool / model / token budgets are user/policy configurable).
- `docs/orchestration.md` §6.1: per-swarm-depth kill switch scope and runaway-swarm
  cancellation semantics.
- `docs/governance.md` §8: **autonomy policy fields** — `max_waves`,
  `max_child_agents_per_wave`, `recursion_enabled`, `max_recursion_depth`,
  `max_wall_clock_minutes`, `max_tool_calls`, `max_model_calls`, `max_token_budget`,
  `external_write_tier_limit`, `auto_candidate_creation_allowed`,
  `min_confidence_to_spawn`. Resolution rule: most-restrictive value wins across
  task-type, tenant, actor, risk-tier, entrypoint, routine, and tool-plan scopes,
  logged as `decision_kind: autonomy_budget_resolved`.
- `docs/versioning.md` §4.3: **candidate lifecycle** with auto-creation, async/non-
  blocking review, archive-before-hard-delete, and retention TTLs
  (`candidate_review_ttl`, `archive_retention_ttl`, `duplicate_candidate_ttl`). Notes
  this layered lifecycle is the mechanism that bounds candidate-variant explosion.
- `docs/operations.md` §6 (added bullets): swarm/autonomy monitoring signals — spawn
  count, recursion depth, wave count, budget-exhaustion rate by which budget tripped
  first, HITL escalation rate by trigger, kill-switch events including runaway-swarm
  cancellations, candidate growth rate by source, archive growth rate by reason.
- `docs/operations.md` §7: garbage-collection policies for duplicate / unreviewed /
  rejected / superseded candidates and for archived artifacts, with the explicit
  archive-before-hard-delete invariant and a Decision-log entry per sweep
  (`decision_kind: gc_swept`).
- `AGENTS.md` §3: glossary entries for **Swarm Supervisor**, **Spawn ledger**,
  **Autonomy budget**, **Recursion depth**.
- `AGENTS.md` §4: new boundary row clarifying that the Swarm Supervisor decides
  whether to spawn; routines/agents/executors may only *request* spawns.
- `AGENTS.md` §5: anti-patterns 11–14 — no unbounded recursive spawning without
  supervisor controls; every spawn must have a ledger entry written before the child
  runs; no direct hard-delete (archive first); auto candidate creation is non-blocking
  unless policy says otherwise; "dynamic spawning is only predefined templates" is
  explicitly identified as incorrect and to be corrected on sight.
- `AGENTS.md` §6.6: two new `grep` checks for stale dynamic-spawning claims and for
  "unbounded recursion".
- `README.md`: Swarm Supervisor surfaced as a first-class control-plane component;
  control-plane Mermaid diagram updated; new §4.1 dynamic-spawning diagram (Slack /
  API / MCP -> Task -> Swarm Supervisor -> waves / recursive children ->
  governance / validator / evaluator -> outputs); repo map updated.
- `examples/autonomy-policy.example.yaml`: a conservative tenant-scoped autonomy
  budget with risk-tier, routine, and actor-trust overrides.
- `examples/spawn-ledger.example.json`: one populated row of the spawn ledger,
  including budgets, risk tier, depth, output contract, and provenance pointers.

### Changed
- `docs/architecture.md` §1, §3.2.1: Swarm Supervisor listed in the control plane,
  with the explicit statement that it does not write Task `state` and does not invoke
  tools directly. The control-plane Mermaid diagram includes the supervisor and its
  consultation edge to Governance.
- `examples/criterion.example.yaml`: added a `lifecycle` block with
  `candidate_review_ttl_days`, `archive_retention_ttl_days`, and
  `archive_before_hard_delete: true` to make the candidate-lifecycle contract
  concrete on an existing example.

### Fixed
- No prior doc claimed "dynamic spawning is only predefined templates", but the
  `grep` check added in §6.6 prevents this regression in future drops.

### Known limitations of v0.1.2
- Still no code, schemas in a typed language, or CI.
- The spawn ledger is described as append-only and tenant-scoped; the storage
  technology is left to the implementation.
- Temporal is named as an implementation candidate for the supervisor's control
  loop, but no Temporal-specific schema is provided — that belongs in the
  implementation repo.
- GraphQL entrypoint remains deferred; multi-region, fine-tuning, and a routine
  marketplace remain out of scope.

## [0.1.1] — 2026-05-15

Documentation patch addressing the second-pass docs QA council. Reference architecture
only — no runnable code. No breaking shape changes; clarifies semantics and tightens
terminology across the docs and examples.

### Added
- `docs/versioning.md` §1: explicit **version reference syntax** (`artifact_id@version`), used consistently across docs and examples.
- `docs/versioning.md` §4.1: **status vs operations** for routines — statuses (`draft`, `candidate`, `active`, `deprecated`, `archived`, `pruned`) are separate from operations (`create`, `recall`, `update`, `pin`, `copy`, `deprecate`, `archive`, `prune`).
- `docs/versioning.md` §4.2: **pin bindings** modeled as `(scope, routine_id) -> routine_id@version` rows, not a status.
- `docs/architecture.md` §2.1/§2.2: V1 entrypoint enum (`slack | api | mcp | internal`) with explicit deferral of `graphql` to post-V1. REST is the command/control API; GraphQL is reserved for later complex query surfaces.
- `docs/governance.md` §2.1: policy **trigger phases** flowchart over the Task lifecycle.
- `docs/governance.md` §7: canonical `evaluator_options.auto_accept_criteria` spelling (with `auto_accept_evaluator_criteria` retained as an alias) and the explicit rule that evaluator-generated criteria stay `candidate` until accepted, then are versioned and reused.
- `docs/evaluator.md` §3/§4: criterion status enum aligned with the routine enum (`candidate` replaces older `proposed` spelling).
- `docs/orchestration.md` §4.1–§4.5: tightened **duplicate detection** — fingerprint fields, tenant scope, storage duration, similarity threshold, terminal-state eligibility, and 24-hour suppression-window behavior; dedup branch added to the orchestration flow diagram.
- `docs/orchestration.md` §5.2: **side-by-side version rollout** diagram for long-running Tasks.
- `docs/versioning.md` §5.1: **release manifest pinning flow** diagram.
- `docs/operations.md` §1.0: **log/provenance flow** diagram.
- `docs/proposals/`: new directory with a `README.md` describing when and how to author cross-cutting docs proposals (referenced from `AGENTS.md`).
- `examples/criterion.example.yaml`: a `candidate` evaluator criterion.
- `examples/hitl-decision.example.json`: a worked HITL approval record carrying `from_state` and `resume_to_state`.
- `examples/logs.example.json`: one sample line from each of the five log streams.
- `examples/dedup-fingerprint.example.json`: a dedup fingerprint with retention, threshold, and eligibility metadata.

### Changed
- `docs/task-contract.md` §2: prose definition of `RECEIVED` added; `AWAITING_HITL` return edges now name both pre-execute (`POLICY_CHECK`) and post-execute (`EVALUATING`) gates; HITL block now records `from_state` so the resume target is unambiguous.
- `docs/task-contract.md` §2.1: explicit rule that evaluator criterion **proposals do not pause the Task** — they create review records asynchronously. The `EVALUATING -> AWAITING_HITL` edge fires only when a policy on evaluation outcomes requires HITL.
- `docs/task-contract.md` §6: `dedup` block added to the Task shape (fingerprint, fields, candidate ids, suppression).
- `docs/governance.md` §3: trust-tier predicates now use **set membership** (`tool.tier in {...}`); explicit note that tiers are an unordered enum and `tool.tier > read_safe` is undefined behavior.
- `docs/governance.md` §6: HITL flow records `hitl.from_state` and resumes there on approve.
- `docs/v1-dogfood-scenarios.md` §6: routine-lifecycle scenario rewritten against the new status/ops/pin-binding model.
- `AGENTS.md`: restructured for agent ingestion — added a repository map (§2), a normative glossary (§3), component-boundary guardrails (§4), and explicit anti-patterns (§5). Added a testing/documentation convention (§6.6) covering `grep` checks for forbidden strings, and the rule that new lifecycle states / entrypoint enum values must enter via `docs/proposals/` first.
- `examples/task.example.json`: added `dedup` block, `hitl.from_state`, and `recalled_routine` provenance entry referencing `rt_tldv_to_monday@2`.
- `examples/policy.example.yaml`: added `evaluator_options.auto_accept_criteria` block; clarifying note that the deterministic prefilter uses set-membership predicates.
- `examples/routine.example.yaml`: removed `status: pinned` overload. Status is now `active`; pinning is expressed as a separate `pin_bindings` entry binding `tenant:tenant_acme` to `rt_tldv_to_monday@2`.
- `examples/release-manifest.example.yaml`: every reference rewritten to `artifact_id@version` form; routines do not carry a `pinned: true` flag — pin bindings are now a separate block on the manifest; unused-prompt/runner inconsistencies removed.
- `examples/tool-plan.example.yaml`: added a header note clarifying `default_pack` vs `client_pack`.
- `README.md`: entrypoint diagram aligned with the V1 enum; repo map updated to include the new examples and `docs/proposals/`.

### Fixed
- Entrypoint enum drift: `entrypoint` in `docs/task-contract.md` no longer lists `graphql` as a valid V1 value.
- Old problematic strings removed: `status: pinned`, `tool.tier > read_safe`, and `graphql` as a V1 entrypoint value.
- `AGENTS.md` reference to `docs/proposals/` no longer points at a nonexistent path — the directory now exists with a `README.md` describing its purpose.

### Known limitations of v0.1.1
- Still no code, schemas in a typed language, or CI.
- GraphQL entrypoint is documented as deferred; no schema sketch yet.
- Multi-region, fine-tuning, and a routine marketplace remain out of scope.

## [0.1.0] — 2026-05-15

Initial documentation drop. Reference architecture only — no runnable code.

### Added
- `README.md` with overview, principles, V1 scope, and a Mermaid architecture diagram.
- `AGENTS.md` agent-ingestible instructions for coding harnesses (Claude Code, Codex, Cursor, Gemini, Aider).
- `docs/architecture.md` — control plane, execution plane, persistence, and entrypoints.
- `docs/task-contract.md` — canonical Task object, lifecycle states, and transitions.
- `docs/governance.md` — policy authoring (NL, document, API), persistence, deterministic vs agentic enforcement, HITL flow.
- `docs/evaluator.md` — criteria as proposals, acceptance by human or policy, versioning, and "wrong-but-schema-valid" detection.
- `docs/orchestration.md` — state ownership, retries/idempotency split, wave-based parallelism, duplicate detection (15 min confirm / 24 h suppression), side-by-side versioning, graceful drain, kill switch, poison-pill DLQ.
- `docs/versioning.md` — git-tagged platform vs DB-persisted routines/policies, routine lifecycle (create/recall/update/pin/copy/deprecate/archive/prune), release manifests pinning policy / prompt / runner / tool plan / contract / evaluator versions.
- `docs/tool-plans.md` — tool trust tiers, tool gateway, credential isolation, tenant/client isolation, default pack and extension model.
- `docs/v1-dogfood-scenarios.md` — three concrete workflows plus governance violation, drift, routine lifecycle, and duplicate-GC scenarios.
- `docs/operations.md` — event/decision/action/validation/evaluation logs, provenance fields, model cascade, no-frontend monitoring via Slack/API/MCP.
- `examples/` — example Task, policy, routine, release manifest, and tool plan.

### Known limitations of v0.1.0
- No code, schemas in a typed language, or CI.
- GraphQL entrypoint is described but intentionally deferred.
- Multi-region, fine-tuning, and a routine marketplace are out of scope.
