# AGENTS.md

This file provides guidance to AI coding agents (Claude Code, Codex, Cursor, Aider, and others that read
`AGENTS.md`) when working in this repository.

## What this is

A **documents-only** workspace for work that spans both Money Across Borders repositories. It holds
discovery reports, specifications and task breakdowns. It contains **no application code**, and has no
build, test or lint commands.

| Repository | Path (relative to this folder) | Role | Agent guidance |
|---|---|---|---|
| `money-across-borders` | `../money-across-borders` | Backend: Express + TypeScript, Clean Architecture/DDD, Postgres | [`../money-across-borders/AGENTS.md`](../money-across-borders/AGENTS.md) |
| `money-across-borders-web` | `../money-across-borders-web` | Frontend: Next.js (a version with breaking API changes; read its AGENTS.md) | [`../money-across-borders-web/AGENTS.md`](../money-across-borders-web/AGENTS.md) |

Before reasoning about either codebase, read that repository's own `AGENTS.md`. It is the source of
truth for architecture, commands and invariants. Don't copy its content into this workspace; link to it
instead.

## Layout

```
README.md                     # human overview + feature index
AGENTS.md / CLAUDE.md         # this guidance
docs/workflows/               # multi-agent procedures (README.md index, tdd.md)
templates/                    # reusable briefs that start each stage
  discovery-brief.md
  spec-definitions-brief.md
features/<feature-slug>/
  README.md                   # status, stage table, spec decisions, open-question owners
  01-discovery/brief.md       # the request (input)
  01-discovery/report.md      # the discovery report (output)
  02-specs/brief.md           # the request (input)
  02-specs/specifications.md  # proposed specs for approval (output)
  03-tasks/                   # implementation tasks, only after specs are approved
```

## Feature lifecycle

Each feature moves through numbered stages. **Don't skip a stage, and don't start the next one until
the user asks for it.**

1. **Discovery:** start from `templates/discovery-brief.md`. Investigate both repos and write
   `01-discovery/report.md`.
2. **Specifications:** start from `templates/spec-definitions-brief.md`. Turn the approved report into
   `02-specs/specifications.md`, keeping feature requirements separate from technical improvements.
3. **Tasks:** only after engineering approves the specs, break them into tasks under `03-tasks/`.
   Each task is then implemented with the [TDD workflow](docs/workflows/tdd.md).

When a stage finishes, update the feature's `README.md` (stage status, spec decisions) and the feature
table in the root `README.md`.

## Workflows

Procedures that several agents follow live in [`docs/workflows/`](docs/workflows/README.md). This file
keeps the global rules; when a workflow disagrees with it, this file wins.

| Work | Workflow |
|---|---|
| Implementing a task from `03-tasks/` | [TDD](docs/workflows/tdd.md): RED (Codex) → GREEN (Claude) → REFACTOR (Claude, optional) → VERIFY (Codex) |

**Before starting any TDD phase, read [`docs/workflows/tdd.md`](docs/workflows/tdd.md) in full and
follow it.** Each phase records its evidence in the task file's `## TDD log` section.

## Rules

- **Discovery and spec stages are read-only for the sibling repositories.** Don't modify source code,
  databases or migrations in `../money-across-borders` or `../money-across-borders-web` unless the user
  explicitly asks for implementation.
- **Don't assume undocumented business rules.** Write them down as open questions instead.
- **Open questions stay open** until the user records an answer. Number them (`Q1`, `Q2`, …), list which
  specs each one blocks, and have acceptance criteria refer to them (e.g. "per Q2") rather than guessing.
- **Back up claims with references** to the sibling repos, as `file_path:line` or a function name, so
  reviewers can check them.
- **Documents that cover both repos go here**, not inside either code repository. (The backend's
  `/task-context` skill writes single-task context under `../money-across-borders/specs/`; that's the
  place for backend-only work.)

## Conventions

- Feature folders use kebab-case slugs (`named-wallets`).
- Each stage uses `brief.md` for the input, plus a descriptive output name (`report.md`,
  `specifications.md`).
- All links are relative, so the workspace can be moved as a whole. After moving or renaming a file,
  check that every link still resolves.
- IDs: feature specs `F1, F2…`, technical-improvement specs `T1, T2…`, open questions `Q1, Q2…`, each
  numbered within its feature. Don't renumber existing IDs; reviewers refer to them.
- Dates are absolute (`2026-10-13`), never relative ("yesterday").

## Git

- Default branch: `main`. Remote: `origin` (GitHub, `JeanRiffel/money-across-borders-workspace`).
- Commit only when the user asks. The one standing exception is the VERIFY-pass step of the
  [TDD workflow](docs/workflows/tdd.md#verify-verifier), which commits the task and opens its PRs.
