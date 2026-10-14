# NW-01: Postgres wallet repository integration tests (baseline)

| | |
|---|---|
| **Spec** | [SPEC-T4](../02-specs/specifications.md#spec-t4-postgres-wallet-repository-integration-tests) |
| **Repo** | `money-across-borders` |
| **Depends on** | – |
| **Status** | Not started |

## Goal

Put `PostgresWalletRepository` under automated test **before** NW-03/NW-04 change its SQL, so those
tasks have a safety net and a place to add `name` assertions.

This is a **characterization** task: the tests describe what the repository does today, so they are
expected to **pass** in RED. If one fails, that is a bug in the current code: log it, and GREEN fixes
it. If all pass, GREEN has nothing to do and the task goes straight to VERIFY.

## Acceptance criteria

1. The tests run against a real, migrated Postgres with a documented command that is **not**
   `npm test` (which must keep needing no services). Either add a new Jest config and script (e.g.
   `jest.db.config.ts` + `npm run test:db`) or extend an existing Postgres suite; document the choice
   in the repo's `AGENTS.md` "Commands" section.
2. `save()` then `findById()` round-trips every column: id, account id, currency, balance, status,
   created at.
3. `save()` then `findByAccountIdAndCurrency()` returns the same wallet; another currency or another
   account returns `null`.
4. Saving an existing wallet again (upsert) updates balance and status, and keeps `id`,
   `account_id`, `currency` and `created_at`.
5. Inserting a second wallet with the same `(account_id, currency)` and a different id is rejected
   by the database (Postgres error code `23505`).
6. Each test uses its own account and cleans up after itself, so it never touches the treasury
   wallets or another test's rows.

## References

- Repository: `src/infra/persistence/postgresql/postgres-wallet-repository.ts` (`save` lines 35–51,
  `SELECT`s at lines 55 and 67).
- Fixture helpers to reuse or copy: `__tests__/concurrency/support/db.ts` (`createLabAccount`,
  `deleteLabAccount`).
- Existing Postgres-only suite and its config: `jest.concurrency.config.ts` (run with
  `npm run db:migrate && npm run test:concurrency`).
- Why it's needed: `docs/known-issues.md:9`.

## Out of scope

- `name` (added by NW-04) and `findByAccountId` (NW-06). Those tasks extend this suite.
- Integration tests for any other repository.

## TDD log

### RED (Codex)

- Date:
- Tests added/changed:
- Command and result (expected: pass; list any failure as a bug):

### GREEN (Claude)

- Date:
- Files changed (or "none, all tests passed in RED"):
- Results of `npm test`, lint and format check:

### REFACTOR (Claude)

- Date:
- Changes and reasons (or "No refactoring needed: <reason>"):
- Results of the full suite, lint and format check; test SHA-256 unchanged:

### VERIFY (Codex)

- Date:
- Result (pass / findings):
