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

## Workflow

Each feature moves through numbered stages. **Don't skip a stage, and don't start the next one until
the user asks for it.**

1. **Discovery:** start from `templates/discovery-brief.md`. Investigate both repos and write
   `01-discovery/report.md`.
2. **Specifications:** start from `templates/spec-definitions-brief.md`. Turn the approved report into
   `02-specs/specifications.md`, keeping feature requirements separate from technical improvements.
3. **Tasks:** only after engineering approves the specs, break them into tasks under `03-tasks/`.

When a stage finishes, update the feature's `README.md` (stage status, spec decisions) and the feature
table in the root `README.md`.

## Implementation (TDD)

Each task in `03-tasks/` is implemented in three steps, each by a fixed agent. One step finishes before
the next starts, and each agent records its result in the task file's `## TDD log` section.

| Step | Agent | Does | Must not |
|---|---|---|---|
| **RED** | Codex | Write tests that encode the task's acceptance criteria, and update existing tests whose setup no longer matches the new contract. Run them and confirm they fail **for the expected reason** (missing behavior, not a typo or broken import). Log the test files and the failing output. | Write or change production code. |
| **GREEN** | Claude | Write the minimum production code that makes the RED tests pass, then run the repo's full suite, lint and format check. Log the files changed. | Edit, delete or skip RED tests. If a test looks wrong, stop and log why; it goes back to RED. |
| **VERIFY** | Codex | Re-run the suite, lint and format check. Check the code against the acceptance criteria and the repo's `AGENTS.md` invariants, and confirm the RED tests are unchanged since RED. Log *pass*, or findings that send the task back to GREEN. | Fix the code itself. |

- A test may only encode a rule that the spec states or an answered open question decides. A task that
  depends on an open question can't enter RED.
- Commands come from each repo's `AGENTS.md` (backend: `npm test`; `npm run test:integration` needs
  Postgres). The frontend has no test runner until SPEC-T5 lands, so frontend tasks wait for it.
- Tasks with no behavior to test first (e.g. a test harness) say so in their task file and skip RED.

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
- Commit only when the user asks.
