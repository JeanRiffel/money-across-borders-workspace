# NW-01: Postgres wallet repository integration tests (baseline)

| | |
|---|---|
| **Spec** | [SPEC-T4](../02-specs/specifications.md#spec-t4-postgres-wallet-repository-integration-tests) |
| **Repo** | `money-across-borders` |
| **Depends on** | – |
| **Status** | Done |

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

- Date: 2026-10-15.
- Branch (`money-across-borders`): `test/nw-01-postgres-wallet-repository`; changes uncommitted.
  Created from updated `main` at `db8be71` after `git checkout main && git pull` (exit 0).
- Test integrity (new RED hash, SHA-256):
  - `__tests__/infra/persistence/postgresql/postgres-wallet-repository.test.ts`:
    `45efd3f963f880200081d4eeceb293c868a5f3257ee1995ff484f4d945cea08f`.
- Files created or changed (backend):
  - Created `__tests__/infra/persistence/postgresql/postgres-wallet-repository.test.ts`: six
    characterization tests against the real repository and Postgres; existing lab fixture helpers reused.
  - Created `jest.db.config.ts`: separate Postgres runner, one worker, 20-second test timeout.
  - Changed `package.json`: added `test:db`; no new dependencies.
  - Changed `jest.config.ts`: excludes the new Postgres directory from service-free `npm test`.
  - Changed `AGENTS.md`, Commands: documented `test:db` and migration prerequisite.
  - Changed `.github/workflows/ci.yml`: runs `test:db` after migrations in the existing Postgres job,
    as required by SPEC-T4 criterion 1 when CI provides Postgres.
- Files changed (workspace): this task and `03-tasks/README.md` for status/evidence;
  feature and root `README.md` for implementation progress.
- Commands and results:
  - `npm run db:migrate` in sandbox: exit 1, `connect EPERM 127.0.0.1:5432`;
    no characterization behavior exercised.
  - `npm run test:db` in sandbox: exit 1, 1 failed suite / 6 failed tests due to the same
    `EPERM` in fixture setup (and cleanup verification), not repository assertion failures.
  - `npm run db:migrate && npm run test:db` with sandbox restriction lifted: exit 0;
    5 migrations applied; 1 suite passed, 6 tests passed, 0 snapshots.
  - `npm test` in sandbox: exit 1; 3 suites failed, 49 passed (52 total);
    20 tests failed, 236 passed (256 total). Failures: local listener `EPERM` in
    `resilient-http-client.test.ts` and `http-exchange-rate-provider.test.ts`, plus timeouts
    (15000 ms and 5000 ms) in `generate-dataset.test.ts` under the parallel run.
    Re-run serially with local sockets permitted, per workflow section 9.
  - `npm test -- --runInBand` with local sockets permitted: exit 0;
    52 suites passed, 256 tests passed, 0 snapshots. No external services required by this suite.
  - `npm run lint`: exit 0, 0 errors, 22 existing warnings in production files.
  - `npm run format:check`: exit 0, all matched production TypeScript files formatted.
  - `npx prettier --check jest.db.config.ts __tests__/infra/persistence/postgresql/postgres-wallet-repository.test.ts`:
    exit 0, both files formatted (the standard format command only covers `src`).
  - `git diff --check` (backend): exit 0.
- Acceptance criteria → coverage (test names in the new file):
  - **1:** `jest.db.config.ts`, `test:db` script, default-suite exclusion, backend Commands
    documentation, CI integration step and successful migrated-Postgres execution above.
  - **2:** `save then findById round-trips every mapped column` compares id, account id,
    currency, nonzero balance, status and a fixed timestamp with millisecond precision.
  - **3:** `findByAccountIdAndCurrency returns the saved wallet`,
    `findByAccountIdAndCurrency returns null for another currency`, and
    `findByAccountIdAndCurrency returns null for another account`.
  - **4:** `upsert updates balance and status while preserving identity and creation columns`.
    Supplies a different valid account, currency and timestamp on the same id so unintended updates
    cannot pass merely because input values were identical; checks persisted original fields,
    changed balance/status and absence under the offered account/currency.
  - **5:** `rejects another id for the same account and currency with Postgres code 23505`;
    also checks the original row remains and the rejected id was not inserted.
  - **6:** every test uses `beforeEach` → `ownAccount` → `createLabAccount` (fresh UUID).
    Additional accounts are tracked too. `afterEach` awaits all scoped `deleteLabAccount` calls
    and queries those exact UUIDs to assert no wallet/account rows remain; `afterAll` closes the pool.
    No test uses treasury ids or unscoped cleanup.
- Findings, blockers or limitations: no repository bug found by the six characterization tests.
  Initial sandbox connection/listener restrictions require reruns with access to local sockets;
  they are environment failures, not bugs in the current repository. No production code changed.

- Next: VERIFY, by Codex. GREEN and REFACTOR skipped because all characterization tests pass;
  no production implementation or refactoring is needed. VERIFY has not been executed in this hand-off.

### GREEN (Claude)

- Date:
- Files changed (or "none, all tests passed in RED"):
- Results of `npm test`, lint and format check:

### REFACTOR (Claude)

- Date:
- Changes and reasons (or "No refactoring needed: <reason>"):
- Results of the full suite, lint and format check; test SHA-256 unchanged:

### VERIFY (Codex)

- Summary: Verified the unchanged RED test against all six acceptance criteria and the backend's
  AGENTS.md and financial invariants. Re-ran migrations, the real-Postgres suite, the full default
  suite, lint and formatting successfully. Characterization passes without production changes;
  GREEN and REFACTOR remain skipped and NW-01 is Done.
- Date: 2026-10-15.
- Branch (`money-across-borders`): `test/nw-01-postgres-wallet-repository`; committed as
  `12544ea773baa6d0d9c7fcb4b8dc95d8e20b6b66` and pushed.
  [Backend PR #58](https://github.com/JeanRiffel/money-across-borders/pull/58).
- Branch (`money-across-borders-workspace`): `docs/nw-01-verify`, created in an isolated worktree
  from updated `origin/main` at `0e50d20`; only this task and three READMEs are included.
- Test integrity: SHA-256 before and after verification matches RED exactly:
  - `__tests__/infra/persistence/postgresql/postgres-wallet-repository.test.ts`:
    `45efd3f963f880200081d4eeceb293c868a5f3257ee1995ff484f4d945cea08f`.
- Files created or changed: no test or production changes during VERIFY. Updated this task,
  `03-tasks/README.md`, `features/named-wallets/README.md` and root `README.md` with Done status
  and verification evidence. Backend commit contains only the six files documented in RED.
- Commands and results (local sockets permitted to avoid the RED sandbox restrictions):
  - `sha256sum __tests__/infra/persistence/postgresql/postgres-wallet-repository.test.ts`:
    exit 0, exact RED match before and after.
  - `npm run db:migrate && npm run test:db`: exit 0; five migrations applied,
    1 suite passed, 6 tests passed, 0 snapshots.
  - `npm test -- --runInBand`: exit 0; 52 suites passed, 256 tests passed, 0 snapshots.
  - `npm run lint`: exit 0; 0 errors, 22 existing warnings.
  - `npm run format:check`: exit 0; all matched `src/**/*.ts` files formatted.
  - `npx prettier --check jest.db.config.ts __tests__/infra/persistence/postgresql/postgres-wallet-repository.test.ts`:
    exit 0; both new TypeScript files formatted.
  - `git diff --check`: exit 0 in backend and workspace documentation worktree.
  - `git commit` (normal pre-commit `npm test`): exit 0; 52 suites passed,
    256 tests passed. No hook bypass.
  - Backend `git push -u origin test/nw-01-postgres-wallet-repository` and `gh pr create`:
    exit 0; PR #58 opened against `main`.
- Acceptance review (backend paths):
  - **1 — pass:** `jest.db.config.ts:1-10`, `package.json:73`, `jest.config.ts:16`,
    `AGENTS.md:105`, `.github/workflows/ci.yml:110-111`: migrated-Postgres command is documented,
    excluded from default tests and wired after migrations in the existing CI Postgres job.
  - **2 — pass:** `__tests__/infra/persistence/postgresql/postgres-wallet-repository.test.ts:68`
    uses the six-field projection at `fields` to compare every contracted column.
  - **3 — pass:** the same test file at lines 74, 82 and 87 checks the matching wallet,
    another currency and another account respectively.
  - **4 — pass:** the same test file at line 95 checks changed balance/status and preserved
    id/account/currency/created-at despite different offered immutable values.
  - **5 — pass:** the same test file at line 119 asserts `23505`, retained original row
    and absent rejected id. This matches `001_init_schema.sql:58` and the repository upsert.
  - **6 — pass:** the same test file at lines 28, 46, 51 and 66 creates fresh accounts,
    tracks extra accounts, awaits cleanup and verifies no scoped rows remain, then closes the pool.
    `__tests__/concurrency/support/db.ts:13-16,37-39` uses generated UUIDs and account-scoped deletes.
- AGENTS.md / invariants review: tests use real existing domain objects and the infra repository;
  no architecture boundary, auth rule, production configuration, migration or guaranteed invariant
  changed. Nonzero fixture balances characterize persistence only; they do not claim ledger funding
  or concurrent-write guarantees. Treasury and unrelated account rows are untouched by test cleanup.
- Findings, blockers or limitations: pass; no V findings. Lint warnings predate this task.
  Hosted CI results are separate from the successful local verification; PR review and merge are
  left to a human. Existing workspace AGENTS.md/workflow edits are excluded from this task's commit.
- Next: Done; human review of both PRs, followed by human-controlled merge.
- For the next agent: read this entry and the RED coverage map first. Claude has no GREEN or
  REFACTOR work for NW-01. On future NW-04 work, extend this suite for `name` in a new RED round
  and record its new hash; do not silently rewrite this baseline. Run `npm run db:migrate &&
  npm run test:db` with local Postgres access, and permit local HTTP listeners for the default suite.
  Do not merge the PRs automatically or include unrelated workspace workflow changes.
