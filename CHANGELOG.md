# Changelog

All notable changes to this reference architecture are documented here.
The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and the project
uses [Semantic Versioning](https://semver.org/) for the **documentation** itself
(the implementing platform will track its own version).

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
