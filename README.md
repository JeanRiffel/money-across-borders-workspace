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

### Conventions

- Feature folders use kebab-case slugs (e.g. `named-wallets`).
- Stage folders are numbered so they sort in workflow order.
- Links between documents are relative, so the folder can be moved as a whole.
- Open questions stay open until someone with authority answers them. Record the answer and the
  owner in the feature's README.

## Features

| Feature | Stage | Status |
|---|---|---|
| [Named Wallets](features/named-wallets/README.md) | 2. Specifications | Awaiting engineering approval |
