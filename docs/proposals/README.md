# Proposals

Short markdown files describing changes that span multiple docs *before* the docs are
edited. Each file is a single-author, time-boxed proposal for a change to the reference
architecture.

This directory is normally empty. A proposal lives here only between drafting and
acceptance — once a proposal is accepted, the change is applied across the affected docs
and the proposal file is deleted in the same commit.

---

## When to write one

Open a proposal here before editing docs when the change:

- Adds, removes, or renames a value in a closed enum (Task state, entrypoint, trust tier, routine status).
- Adds or moves a component boundary (Governance vs Validator vs Evaluator vs Routine).
- Changes the Task contract shape (a new required field, a removed field, a renamed field).
- Changes how versioning works (status enum, pin binding model, manifest resolution).

Do **not** open a proposal for typos, formatting, example tweaks, or single-doc edits.

---

## File format

```
docs/proposals/<short-kebab-slug>.md
```

A proposal must contain:

1. **Title.** One line, what is changing.
2. **Motivation.** Two or three sentences.
3. **Affected docs and examples.** Bulleted list of file paths.
4. **Proposed change.** The actual edit, in enough detail that a reader can predict the diff.
5. **Migration story.** What happens to existing references to the old form.
6. **Open questions.** Optional.

Keep proposals under ~150 lines. If a proposal grows past that, the change is probably
too large to land in one commit and should be split.

---

## Lifecycle

```
draft -> accepted -> applied (file deleted) | rejected (file deleted)
```

A proposal is **applied** in the same commit as the docs changes it describes. The
commit message references the proposal by slug.
