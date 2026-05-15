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

## 2. Boundaries (do not cross without explicit instruction)

1. **Do not invent new top-level concepts.** If you find a gap, propose it in a doc, do not silently extend the model.
2. **Do not change the Task contract shape** in `docs/task-contract.md` without simultaneously updating: `architecture.md`, `orchestration.md`, `evaluator.md`, and `examples/task.example.json`.
3. **Do not weaken governance.** Policies are durable. Criteria are proposals until accepted. Never document a path where an agent bypasses policy enforcement.
4. **Do not expose raw credentials to agents.** Tool calls go through the Tool Gateway. Document accordingly.
5. **Do not introduce a frontend.** V1 monitoring is Slack / API / MCP only.
6. **Do not bundle V1 with later features.** GraphQL, marketplace, multi-region, fine-tuning are explicitly deferred.

---

## 3. Non-goals (for V1 documentation)

- Production deployment manifests (Terraform, Helm, etc.).
- Exact DB schemas. The docs describe the **shape** of records, not DDL.
- Vendor lock-in to a specific LLM provider. The "model cascade" is a tier description, not a model list.
- A web UI.
- Performance SLOs. State the latency/consistency *concerns* but do not invent numbers.

---

## 4. Implementation conventions for this repo

When you edit this repository:

### 4.1 File layout
- All long-form docs live in `docs/`.
- Examples live in `examples/` as `.json` or `.yaml`. Keep them small and illustrative.
- `README.md`, `CHANGELOG.md`, and `AGENTS.md` stay at the root.

### 4.2 Markdown style
- One `# H1` per file, matching the filename's intent.
- Section numbering is allowed (e.g. `## 3.`) when a doc has more than three top-level sections; otherwise omit.
- Use Mermaid for diagrams. Prefer `flowchart` and `stateDiagram-v2`. Avoid sequence diagrams longer than ~15 lines.
- Tables are encouraged for enumerations (states, tiers, scenarios).
- Keep paragraphs short. Bias to bullet lists where the content is a list.

### 4.3 Terminology (use these exact terms)
- **Task** (capitalized) — the canonical work-unit object.
- **Orchestrator** — owns state transitions.
- **Planner / Decomposer** — optional split; treat as one concept unless splitting matters.
- **Governance** / **Policy** — durable rules; **Policy Engine** enforces.
- **Evaluator** — produces evaluation results and proposes criteria.
- **Criteria** — what an evaluator measures; proposals until accepted.
- **Routine** — a reusable, DB-persisted execution pattern.
- **Release manifest** — pins versions of policy, prompt, runner/routine, tool plan, contract, evaluator criteria.
- **Tool Gateway** — credential boundary and tool-call enforcement point.
- **Tool plan** — per-tenant, per-client bundle of allowed tools and trust tiers.
- **Wave** — a parallel batch of sibling subtasks.
- **HITL** — Human-In-The-Loop approval step.
- **DLQ** — Dead Letter Queue for poison-pill messages.

### 4.4 Cross-references
- When introducing a concept, link to the doc that defines it: e.g. `See [governance.md](docs/governance.md).`
- Do not duplicate definitions across docs — link instead.

### 4.5 Examples
- Examples are **illustrative**, not normative. State this at the top of each example file with a comment.
- Use ISO-8601 UTC timestamps in examples: `2026-05-15T10:33:00Z`.
- Use fake but plausible IDs: `task_01HXYZ...`, `pol_001`, `rt_001`.

---

## 5. Safety rules

These apply both to what you **document** and to what you **do** in this repo.

1. **No secrets in repo.** Never include real tokens, API keys, or credentials in examples. Use placeholders like `${MONDAY_API_TOKEN}`.
2. **No PII in examples.** Use fake names, emails (`user@example.com`), and org IDs.
3. **No destructive git operations** without explicit user instruction (`reset --hard`, `push --force`, branch deletion).
4. **Do not auto-approve PRs.** You are not authorized to approve. You may open and comment.
5. **Respect tenant isolation in all examples.** Every example object that has tenant scope must include a `tenant_id` field.
6. **Do not introduce telemetry that violates the logged-provenance contract.** Every action must be attributable.

---

## 6. How to extend the architecture safely

A typical extension looks like this:

1. Identify which doc(s) own the concept.
2. If it spans multiple docs, draft a short proposal at the top of `docs/architecture.md` under a "Pending changes" section, or in a dedicated `docs/proposals/` file if multiple are in flight.
3. Update the affected docs in one commit.
4. Add or update one or more examples in `examples/`.
5. Add a `CHANGELOG.md` entry under an `## [Unreleased]` heading (create it if missing).

Do **not** ship a doc change that:
- Renames a contract field without updating examples.
- Adds a state to the Task lifecycle without updating `orchestration.md`.
- Adds a new entrypoint without updating `architecture.md` and `README.md`.

---

## 7. How to implement this architecture (in a separate code repo)

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

## 8. Conflicts between this file and the user's request

If the user's instruction directly contradicts a rule in this file:

- For **safety rules (§5)**: refuse and explain.
- For **boundaries (§2)**: ask once; if the user confirms, proceed but record the deviation in the commit message.
- For **conventions (§4)**: follow the user's preference for this change.

---

## 9. Where to look next

- Architecture overview: [docs/architecture.md](docs/architecture.md)
- Task object: [docs/task-contract.md](docs/task-contract.md)
- V1 scope: [docs/v1-dogfood-scenarios.md](docs/v1-dogfood-scenarios.md)
- Governance and evaluation: [docs/governance.md](docs/governance.md), [docs/evaluator.md](docs/evaluator.md)
- State machine and concurrency: [docs/orchestration.md](docs/orchestration.md)
- Versioning model: [docs/versioning.md](docs/versioning.md)
- Tool plans and credentials: [docs/tool-plans.md](docs/tool-plans.md)
- Operations / monitoring: [docs/operations.md](docs/operations.md)
