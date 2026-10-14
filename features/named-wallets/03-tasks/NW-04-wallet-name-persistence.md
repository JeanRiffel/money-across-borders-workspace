# NW-04: Migration 006, database constraints and seed writers

| | |
|---|---|
| **Spec** | [SPEC-F1](../02-specs/specifications.md#spec-f1-wallet-name-on-creation-backend--database), [SPEC-T4](../02-specs/specifications.md#spec-t4-postgres-wallet-repository-integration-tests) |
| **Repo** | `money-across-borders` |
| **Depends on** | NW-01, NW-03 |
| **Decisions** | Q3, Q4, Q8 |
| **Status** | Not started |

## Goal

Create the `wallets.name` column, backfill existing rows, and enforce the rules in the database, so
they hold under concurrency and for every writer, not only the API.

## Acceptance criteria

Migration `src/infra/persistence/postgresql/migrations/006_add_wallet_name.sql`:

1. Adds `name` to `wallets`. Every existing wallet, treasury included, gets `"<CURRENCY> Wallet"`
   (e.g. `"USD Wallet"`). The column is then `NOT NULL`.
2. Adds a case-insensitive unique index on `(account_id, lower(name))`, named
   **`uq_wallets_account_lower_name`** (the controller test in criterion 6 relies on the name).
3. Is idempotent: `npm run db:migrate` succeeds when run twice in a row on a fresh database, and on
   a database that already has wallets (the runner re-applies every file each time, discovery R8).
4. Any `CHECK` on `name` must not reject a name that `WalletName` accepts. `char_length` counts
   code points, not graphemes, so at most check that the name isn't blank.

Database behavior (extend the NW-01 suite):

5. Every column, `name` included, round-trips through `save()` → `findById()` /
   `findByAccountIdAndCurrency()`, and the upsert keeps the stored name.
6. Two wallets on the same account with names that differ only in case are rejected by the
   database (`23505`, constraint `uq_wallets_account_lower_name`). The same name on two different
   accounts is accepted.
7. When two requests race and the database rejects the second with that constraint,
   `OpenWalletController` returns **409 with the duplicate-name message** from NW-03, not the
   current "already has a wallet in <currency>" message. A `23505` on the currency constraint keeps
   today's message. (Unit-testable with a fake error carrying `code` and `constraint`.)

Other writers:

8. `src/infra/seed/persistence/batch-writer.ts` (`insertWallets`, ~line 82) and
   `src/infra/seed/types.ts:28` write a name that is unique per account (the backfill format works,
   since accounts have one wallet per currency). `npm run test:seed` passes.
9. `__tests__/concurrency/support/db.ts` (`createLabWallet`) writes a name, and
   `npm run test:concurrency` passes.

## Notes for GREEN

- **Probable trap:** `002_seed_treasury_wallets.sql` runs again on every migration run and inserts
  without `name`. Postgres checks `NOT NULL` before `ON CONFLICT DO NOTHING`, so once 006 has made
  the column `NOT NULL`, the second run of 002 will probably fail. Criterion 3 catches this. Fix it
  without breaking a fresh database, where 002 runs before the column exists.

## References

- Schema: `migrations/001_init_schema.sql:49` (`wallets`), `004_add_wallet_version.sql` (pattern for
  an additive, idempotent column).
- Runner: `migrations/run-migrations.ts`.
- Current 23505 fallback: `src/interfaces/http/controllers/open-wallet.controller.ts:13-21, 51-56`.

## Out of scope

- `findByAccountId` (NW-06).

## TDD log

### RED (Codex)

- Date:
- Tests added/changed:
- Command and failing output (summary):

### GREEN (Claude)

- Date:
- Files changed:
- Results of `npm test`, `test:db` (or the NW-01 command), `test:seed`, `test:concurrency`, lint and format check:

### VERIFY (Codex)

- Date:
- Result (pass / findings):
