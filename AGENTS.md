# AGENTS.md — Instructions for Coding Agents

This file is the contract between this reference architecture and any coding agent that
implements or extends it (Claude Code, Codex, Cursor, Gemini, Aider, etc.).

Read this file first. The rest of `docs/` is your specification.

---

## 1. What this repository is, and is not

This repository is **documentation**. It is not the implementation.

- **Is:** a normative description of the agentic mesh — entrypoints, control plane, governance, evaluator, orchestration, versioning, tool plans, V1 dogfood scenarios.
- **Is not:** a Python/TypeScript/Go project. There is no `package.json`, `pyproject.toml`, or `Cargo.toml`, and there should not be one unless explicitly requested.

When asked to "implement" something here:

- If the request is to **edit docs**, edit docs and update `CHANGELOG.md`.
- If the request is to **scaffold code**, scaffold it in a sibling implementation repository, not this one — unless the user explicitly says "add code to this reference repo".

---

## 2. Repository map

```
.
├── README.md                       # entrypoint overview, V1 scope, full diagram
├── CHANGELOG.md                    # docs release notes (Keep a Changelog / semver)
├── AGENTS.md                       # this file
├── docs/
│   ├── architecture.md             # planes, entrypoints, components
│   ├── task-contract.md            # the Task object, state machine, provenance
│   ├── governance.md               # policies, enforcement, HITL, trigger phases
│   ├── evaluator.md                # criteria, proposal-to-accept flow
│   ├── orchestration.md            # state ownership, waves, dedup, kill switch, DLQ
│   ├── versioning.md               # @version syntax, status vs ops, pin bindings, release manifests
│   ├── tool-plans.md               # trust tiers, credentials, default vs client packs
│   ├── v1-dogfood-scenarios.md     # the three workflows + four cross-cutting scenarios
│   └── operations.md               # five log streams, monitoring without a frontend
├── examples/
│   ├── task.example.json           # a populated Task with provenance and dedup
│   ├── policy.example.yaml         # a hybrid HITL policy with evaluator_options
│   ├── routine.example.yaml        # a routine version + pin binding
│   ├── criterion.example.yaml      # an evaluator-proposed criterion
│   ├── hitl-decision.example.json  # an approval record
│   ├── logs.example.json           # one sample line from each log stream
│   ├── dedup-fingerprint.example.json
│   ├── release-manifest.example.yaml
│   └── tool-plan.example.yaml
└── docs/proposals/                 # short proposals for cross-cutting changes (created on demand)
```

If `docs/proposals/` does not yet exist, create it the first time you need it. It is for
short markdown files that describe a change spanning multiple docs *before* the docs are
edited — see §7.

---

## 3. Glossary

Use these terms exactly. Casing matters.

| Term | Meaning |
|------|---------|
| **Task** | The canonical work-unit object. Capitalized. See [docs/task-contract.md](docs/task-contract.md). |
| **Orchestrator** | Sole writer of Task `state`. |
| **Planner / Decomposer** | Produces a plan (waves of subtasks) from a Task. Optional split. |
| **Wave** | A parallel batch of sibling subtasks. |
| **Governance** | The policy layer that decides *whether* a Task or step is allowed. |
| **Policy** | A durable, versioned governance rule. |
| **Policy Engine** | Enforces policies, deterministically or via an LLM judge. |
| **Evaluator** | Scores Task/wave output against criteria. May propose new criteria. |
| **Criterion** | The unit of evaluation. Versioned. Proposed (status `candidate`) until accepted. |
| **Criteria suite** | A versioned bundle of criteria attached to a routine, policy, or release manifest. |
| **Validator (Contract Validator)** | Checks **shape**, not meaning. Cheap, deterministic, ubiquitous. |
| **Routine** | A reusable, DB-persisted execution pattern. Versioned. |
| **Release manifest** | The pinning record. Resolves every dependency a Task runs against. |
| **Pin binding** | A `(scope, routine_id) -> routine_id@version` row. Not a status. |
| **Tool plan** | A per-tenant bundle of allowed tools, tiers, and credential aliases. |
| **Tool Gateway** | The single boundary where credentials are resolved at call time. |
| **Trust tier** | An unordered enum: `read_safe`, `read_sensitive`, `write_revocable`, `write_destructive`, `external_egress`. |
| **HITL** | Human-In-The-Loop approval. Pauses a Task; resumes to the state the gate fired from. |
| **DLQ** | Dead Letter Queue for poison-pill messages. |
| **Provenance** | An append-only list of references that explain *why* a Task did what it did. |
| **Entrypoint** | One of `slack`, `api`, `mcp`, `internal` (V1). `graphql` is reserved for post-V1. |
| **`artifact_id@version`** | The mandatory reference syntax. See [docs/versioning.md §1](docs/versioning.md). |

---

## 4. Component boundary guardrails

These boundaries are load-bearing. Crossing them silently breaks the audit chain.

| Boundary | Who decides | Who must **not** decide |
|----------|-------------|-------------------------|
| **Whether** a Task / step is allowed | **Governance / Policy Engine** | Routines, evaluator, executors. |
| **Shape correctness** of inputs/outputs | **Contract Validator** | Evaluator, governance. (Shape failures revert state; meaning failures emit verdicts.) |
| **Meaning correctness** of outputs | **Evaluator** | Validator (it only sees shape), governance (it decides *allowance*, not *quality*). |
| **State transitions** on a Task | **Orchestrator** | Everyone else. Other components *recommend* transitions. |
| **Routine lifecycle** (status + ops) | **Routine registry**, governed by [docs/versioning.md](docs/versioning.md) | Evaluator (it may propose; it does not promote). Policies (they may govern recall; they do not write rows). |
| **Pinning** | **Pin-binding table**, scoped per §4.2 of versioning.md | The routine itself — pinning is not a status. |
| **Credential resolution** | **Tool Gateway** | Agents, executors, prompts. Agents only ever see the credential **alias**. |

If you find yourself documenting a path where one of these boundaries is crossed, stop
and propose a docs update first.

---

## 5. Anti-patterns

Do not:

1. **Invent new Task lifecycle states.** The set in [docs/task-contract.md §2](docs/task-contract.md) is closed. Add one only via a proposal under `docs/proposals/`.
2. **Invent new entrypoint enum values.** V1 is `slack | api | mcp | internal`. `graphql` is reserved. Anything else needs a proposal.
3. **Use ordinal comparisons on trust tiers.** `tool.tier > read_safe` is undefined. Use set membership: `tool.tier in {"write_revocable", "write_destructive"}`. See [docs/governance.md §3](docs/governance.md).
4. **Use `status: pinned` on a routine.** Pinning is a binding, not a status. See [docs/versioning.md §4](docs/versioning.md).
5. **Pause a Task on a *proposed* evaluator criterion.** Criterion proposals are written asynchronously; they do not transition Task state. See [docs/task-contract.md §2.1](docs/task-contract.md).
6. **Treat the Evaluator as a governance layer.** Criteria do not block tool access; policies do.
7. **Pass credentials through agent context.** Tool calls go via the Gateway. Agents see the alias only.
8. **Introduce a frontend.** V1 monitoring is Slack / API / MCP only.
9. **Bundle V1 with later features.** GraphQL, marketplace, multi-region, fine-tuning are explicitly deferred.
10. **Edit a versioned artifact in place.** All writes are new rows (`@n+1`). The persistence layer enforces this.

---

## 6. Conventions

### 6.1 File layout
- All long-form docs live in `docs/`.
- Examples live in `examples/` as `.json` or `.yaml`. Keep them small and illustrative.
- `README.md`, `CHANGELOG.md`, and `AGENTS.md` stay at the root.
- Cross-doc proposals live in `docs/proposals/<short-slug>.md`.

### 6.2 Markdown style
- One `# H1` per file, matching the filename's intent.
- Section numbering is allowed (e.g. `## 3.`) when a doc has more than three top-level sections; otherwise omit.
- Use Mermaid for diagrams. Prefer `flowchart` and `stateDiagram-v2`. Avoid sequence diagrams longer than ~15 lines.
- Tables are encouraged for enumerations (states, tiers, scenarios).
- Keep paragraphs short. Bias to bullet lists where the content is a list.

### 6.3 Terminology
- Use the glossary in §3 verbatim. If you need a new term, add it to the glossary in the same commit.
- Use the `artifact_id@version` form whenever referring to a specific version.

### 6.4 Examples
- Examples are **illustrative**, not normative. State this at the top of each example file with a comment.
- Use ISO-8601 UTC timestamps: `2026-05-15T10:33:00Z`.
- Use fake but plausible IDs: `task_01HXYZ...`, `pol_001`, `rt_001`.
- Every tenant-scoped example object must include a `tenant_id` field.

### 6.5 Cross-references
- When introducing a concept, link to the doc that defines it.
- Do not duplicate definitions across docs — link instead.

### 6.6 Testing and documentation convention
This repo has no test suite (it is documentation). The equivalent gates are:

1. **`grep` for forbidden strings** before committing. The CI-equivalent checklist:
   - `grep -rn "status: pinned" docs/ examples/` → must return empty.
   - `grep -rn "tool\.tier *> *read_safe" docs/` → must return empty.
   - `grep -rn "\"graphql\"" examples/` → must return empty (the string `graphql` may appear in docs as a deferred-entrypoint reference only).
   - `grep -rn "docs/proposals" .` → if matched, the directory must exist.
2. **Every doc change touches its examples.** If you add a Task field, update `examples/task.example.json`. If you add a routine op, update `examples/routine.example.yaml`.
3. **Every new state, enum value, or top-level concept gets a `docs/proposals/<slug>.md` entry first.** That file is a short markdown describing the change, the affected docs, and the migration story. After review, the proposal is merged into the affected docs and the proposal file is deleted in the same commit.
4. **`CHANGELOG.md` entry** under `## [Unreleased]` for any change that is not a typo or formatting fix.

---

## 7. How to extend the architecture safely

A typical extension looks like this:

1. Identify which doc(s) own the concept.
2. If it spans multiple docs (or introduces a new state, enum, or boundary), draft a short file at `docs/proposals/<slug>.md` first. Create `docs/proposals/` if missing.
3. Update the affected docs in one commit (and delete the proposal file in the same commit once accepted).
4. Add or update one or more examples in `examples/`.
5. Add a `CHANGELOG.md` entry under an `## [Unreleased]` heading (create it if missing).

Do **not** ship a doc change that:
- Renames a contract field without updating examples.
- Adds a state to the Task lifecycle without updating `orchestration.md`.
- Adds a new entrypoint without updating `architecture.md` and `README.md`.
- Adds a new status to the routine enum without updating `versioning.md` and `routine.example.yaml`.

---

## 8. Safety rules

These apply both to what you **document** and to what you **do** in this repo.

1. **No secrets in repo.** Never include real tokens, API keys, or credentials in examples. Use placeholders like `${MONDAY_API_TOKEN}`.
2. **No PII in examples.** Use fake names, emails (`user@example.com`), and org IDs.
3. **No destructive git operations** without explicit user instruction (`reset --hard`, `push --force`, branch deletion).
4. **Do not auto-approve PRs.** You are not authorized to approve. You may open and comment.
5. **Respect tenant isolation in all examples.** Every example object that has tenant scope must include a `tenant_id` field.
6. **Do not introduce telemetry that violates the logged-provenance contract.** Every action must be attributable.

---

## 9. How to implement this architecture (in a separate code repo)

If you are reading this from inside a future implementation repository, follow this order:

1. **Task contract first.** Implement the Task object, its persistence, and a no-op Orchestrator that can transition states.
2. **One entrypoint.** Start with the REST API. Slack and MCP are clients of it.
3. **Tool Gateway with one tier.** Implement `read_safe` tier first; add tiers as policies require.
4. **Governance as a pass-through.** Stub deterministic policies (allow-all) before adding agentic enforcement.
5. **One executor.** A synchronous, single-step executor before introducing waves.
6. **Logging from day one.** Event / decision / action / validation / evaluation logs are not optional.
7. **Evaluator last.** It is the highest-value component, but only after the loop closes end-to-end.
8. **Routines after a workflow runs twice.** Do not pre-build a routine system.

For each milestone, write the smallest test that proves the loop closes.

---

## 10. Conflicts between this file and the user's request

If the user's instruction directly contradicts a rule in this file:

- For **safety rules (§8)**: refuse and explain.
- For **boundaries (§4) or anti-patterns (§5)**: ask once; if the user confirms, proceed but record the deviation in the commit message *and* open a `docs/proposals/` file describing the deviation.
- For **conventions (§6)**: follow the user's preference for this change.

---

## 11. Where to look next

- Architecture overview: [docs/architecture.md](docs/architecture.md)
- Task object: [docs/task-contract.md](docs/task-contract.md)
- V1 scope: [docs/v1-dogfood-scenarios.md](docs/v1-dogfood-scenarios.md)
- Governance and evaluation: [docs/governance.md](docs/governance.md), [docs/evaluator.md](docs/evaluator.md)
- State machine and concurrency: [docs/orchestration.md](docs/orchestration.md)
- Versioning model: [docs/versioning.md](docs/versioning.md)
- Tool plans and credentials: [docs/tool-plans.md](docs/tool-plans.md)
- Operations / monitoring: [docs/operations.md](docs/operations.md)
