# Money Across Borders: Workspace

Home for work that spans both repositories:

- [`money-across-borders`](../money-across-borders): backend (Node/TypeScript, Clean Architecture/DDD, Postgres)
- [`money-across-borders-web`](../money-across-borders-web): frontend (Next.js)

This folder holds documents only: discovery reports, specifications and task breakdowns. Code
changes go in the repository they belong to.

## Layout

```
money-across-borders-workspace/
├── README.md                 # this file
├── AGENTS.md / CLAUDE.md     # guidance for AI agents (CLAUDE.md imports AGENTS.md)
├── docs/workflows/           # multi-agent procedures, e.g. TDD
├── templates/                # reusable briefs to start a new feature
└── features/
    └── <feature-slug>/
        ├── README.md         # status, links, decisions, open-question owners
        ├── 01-discovery/     # brief.md → report.md
        ├── 02-specs/         # brief.md → specifications.md
        └── 03-tasks/         # implementation tasks, after specs are approved
```

## Workflow

1. **Discovery:** copy [`templates/discovery-brief.md`](templates/discovery-brief.md) to
   `features/<feature-slug>/01-discovery/brief.md`, fill it in, and produce `report.md`.
   Read-only: no code or database changes.
2. **Specifications:** copy [`templates/spec-definitions-brief.md`](templates/spec-definitions-brief.md)
   to `02-specs/brief.md`, and produce `specifications.md` for engineering approval.
3. **Tasks:** once specs are approved, break them into implementation tasks in `03-tasks/`.
4. **Implementation:** each task is built with the [TDD workflow](docs/workflows/tdd.md) (see below).

### Engineering workflows

Procedures shared by several agents live in [`docs/workflows/`](docs/workflows/README.md); AGENTS.md
points agents to them and keeps the global rules.

- **Which workflow applies:** every task in a feature's `03-tasks/` follows the
  [TDD workflow](docs/workflows/tdd.md): RED (Codex writes failing tests) → GREEN (Claude makes them
  pass) → REFACTOR (Claude, optional) → VERIFY (Codex checks independently). Tooling tasks skip RED;
  characterization tasks expect RED to pass. The task file says which kind it is.
- **How it connects:** the approved spec defines the rules; the task turns them into acceptance
  criteria; the workflow turns those into tests and code. A test may only encode what the spec or an
  answered open question says.
- **Where the evidence is:** each task file's `## TDD log` holds one entry per phase and per round
  (commands, results, test SHA-256 hashes, findings). See
  [NW-02](features/named-wallets/03-tasks/NW-02-wallet-name-domain.md) for a complete example.

### Conventions

- Feature folders use kebab-case slugs (e.g. `named-wallets`).
- Stage folders are numbered so they sort in workflow order.
- Links between documents are relative, so the folder can be moved as a whole.
- Open questions stay open until someone with authority answers them. Record the answer and the
  owner in the feature's README.

## Features

| Feature | Stage | Status |
|---|---|---|
| [Named Wallets](features/named-wallets/README.md) | 3. Tasks | Implementation in progress; NW-01 and NW-02 Done |
