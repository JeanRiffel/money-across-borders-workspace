# NW-03: `Wallet` carries a name; `POST /wallets` accepts and returns it

| | |
|---|---|
| **Spec** | [SPEC-F1](../02-specs/specifications.md#spec-f1-wallet-name-on-creation-backend--database) |
| **Repo** | `money-across-borders` |
| **Depends on** | NW-02 |
| **Decisions** | Q1, Q2, Q4, Q8 |
| **Status** | Not started |

## Goal

Make the name part of the wallet and of the creation contract, end to end on the in-memory stack
that `npm test` runs. The database side (migration, constraints) is NW-04.

**Breaking change:** `name` becomes required (Q1). See the release note in [README](README.md#release).

## Acceptance criteria

Entity:

1. `Wallet` holds a `WalletName` and exposes it (`getName()`). The constructor takes it as a new
   parameter.
2. `credit()` and `debit()` return wallets with the **same name** (discovery risk R1).

Creation (`POST /wallets` → `OpenWalletUseCase`):

3. A valid `name` returns 201, and `result.name` is the **normalized** name (e.g. `"  My  USD "` →
   `"My USD"`).
4. A missing `name`, or a `name` that isn't a string, returns 400 and creates no wallet.
5. A name that breaks the NW-02 rules returns 400 with the rule's message and creates no wallet.
6. A name already used by another wallet of the **same account**, compared case-insensitively,
   returns **409** with a message that names the conflict (e.g. "Account … already has a wallet
   named …"). This holds whether that wallet is active or closed (Q4). The same name on a
   **different** account is accepted.
7. When the currency is also a duplicate, the existing currency 409 still wins (the currency check
   runs first), so current behavior and messages don't change.
8. After a remittance debits the sender wallet and credits the recipient wallet, both still have
   their names (in-memory repositories).

Everything else that builds wallets:

9. The in-memory treasury wallets get the same names as the migration backfill (Q3, Q8):
   `"USD Wallet"`, `"BRL Wallet"`, `"EUR Wallet"`, `"GBP Wallet"`.
10. `PostgresWalletRepository` reads and writes `name` (row type, mapper, `INSERT`, both
    `SELECT`s) and implements the new uniqueness lookup with `lower(name)`. It is exercised against
    a real database in NW-04, where the column is created.
11. Swagger: `name` is in `OpenWalletInput` (and its `required` list) and in `OpenWalletOutput`,
    and the 409 description mentions duplicate names. Bruno `Open Wallet` requests send an example
    name.

## Notes for RED

- Existing tests that open wallets will start failing because `name` is now required. Update them
  as part of RED (per the [TDD workflow](../../../docs/workflows/tdd.md#red-test-author)), including
  `__tests__/domain/entities/wallet.test.ts` (`buildWallet()`, line 13),
  `__tests__/application/use-cases/wallet/open-wallet-use-case.test.ts`,
  `__tests__/application/wallet/dto/open-wallet-input.test.ts`, and the remittance use-case and
  idempotency tests that open wallets.
- The uniqueness check needs a repository lookup; RED's tests define the method name, GREEN
  implements it in both repositories.

## References

- Entity: `src/domain/wallet/entities/wallet.ts` (constructor line 9, `credit` 44, `debit` 56).
- DTOs: `src/application/wallet/dto/open-wallet-input.ts:7`, `open-wallet-output.ts`.
- Use case: `src/application/wallet/uses-cases/open-wallet-use-case.ts:20`.
- Controller error mapping: `src/interfaces/http/controllers/open-wallet.controller.ts:38-58`.
- Repository interface: `src/domain/wallet/repository/wallet-repository.ts`.
- Treasury seed: `src/infra/persistence/in-memory/seed-treasury-wallets.ts:26`.
- Swagger: `src/interfaces/http/docs/swagger.ts:103-128`. Bruno: `bruno-collection/Wallet/`.

## Out of scope

- Migration, DB constraints and the race between two concurrent requests (NW-04).
- Ownership (NW-05). Listing (NW-06).

## TDD log

### RED (Codex)

- Date:
- Tests added/changed:
- Command and failing output (summary):

### GREEN (Claude)

- Date:
- Files changed:
- Results of `npm test`, lint and format check:

### REFACTOR (Claude)

- Date:
- Changes and reasons (or "No refactoring needed: <reason>"):
- Results of the full suite, lint and format check; test SHA-256 unchanged:

### VERIFY (Codex)

- Date:
- Result (pass / findings):
