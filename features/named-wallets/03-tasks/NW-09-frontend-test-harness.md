# NW-09: Frontend test harness

| | |
|---|---|
| **Spec** | [SPEC-T5](../02-specs/specifications.md#spec-t5-frontend-test-harness) |
| **Repo** | `money-across-borders-web` |
| **Depends on** | – |
| **Status** | Not started |

## Goal

Give the frontend a test runner, so NW-10 and NW-11 can follow the RED → GREEN flow. Today the repo
has no test files and no `test` script (`package.json` has only `dev`, `build`, `start`, `lint`).

**No RED step:** this task is tooling, with no behavior to test first. Claude implements it (GREEN)
and Codex verifies it.

## Acceptance criteria

1. A test runner suited to Next.js 16.3.5. Read the bundled guide first, per the repo's
   `AGENTS.md`: `node_modules/next/dist/docs/` has testing guides for Vitest and Jest. Prefer
   Vitest + React Testing Library + jsdom unless the guide says otherwise.
2. `npm test` runs once and exits (no watch mode by default) and passes.
3. First tests, describing **current** behavior:
   - `openWalletSchema` (`lib/schemas/wallet.ts`): valid input; invalid account id; unsupported
     currency; negative balance; string balance coerced to a number.
   - `OpenWalletForm` (`components/forms/open-wallet-form.tsx`): invalid input shows an inline
     error and sends no request; valid input POSTs to `/api/wallets` (fetch mocked) and shows the
     success card.
4. The repo's README or `AGENTS.md` explains how to run the tests.
5. `npm run lint` and `npm run build` still pass.

## Out of scope

- End-to-end browser tests.
- The name field (NW-10 changes these tests).

## TDD log

### RED

Skipped: tooling task.

### GREEN (Claude)

- Date:
- Files changed:
- Results of `npm test`, `npm run lint`, `npm run build`:

### REFACTOR (Claude)

- Date:
- Changes and reasons (or "No refactoring needed: <reason>"):
- Results of the full suite, lint and format check; test SHA-256 unchanged:

### VERIFY (Codex)

- Date:
- Result (pass / findings):
