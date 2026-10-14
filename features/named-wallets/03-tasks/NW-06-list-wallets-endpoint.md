# NW-06: `GET /wallets`

| | |
|---|---|
| **Spec** | [SPEC-F3](../02-specs/specifications.md#spec-f3-list-wallets-with-names) (backend part) |
| **Repo** | `money-across-borders` |
| **Depends on** | NW-04, NW-05 |
| **Decisions** | Q5, Q6, Q9 |
| **Status** | Not started |

## Goal

Let customers read their wallets, with names, at any time. This is the first read endpoint of the
wallet context.

The account comes from the token (as in NW-05), so the route has **no path parameter**
(`GET /wallets`, not `/accounts/:id/wallets`), and NW-07 isn't a prerequisite.

## Acceptance criteria

1. `GET /wallets` with a valid token returns **200** and
   `{ statusCode: 200, result: [ … ] }`, one item per wallet of the caller's account, each with
   `walletId, accountId, name, currency, balanceMinorUnits, status, createdAt` (same field names
   and formats as `OpenWalletOutput`).
2. Closed wallets are included, with their status.
3. Items are ordered by `createdAt` ascending, then by currency. *(Proposal made while writing the
   task; change it here before RED if you want another order.)*
4. An account with no wallets returns `result: []`.
5. No token or an invalid token returns 401.
6. Only the caller's wallets are returned: wallets of other accounts, including the treasury
   account, never appear.
7. `WalletRepository.findByAccountId` exists in both repositories. The Postgres implementation is
   covered by the NW-01 suite (returns every wallet of the account, none of another account's).
8. Swagger documents the route and a list response schema. Bruno gets a "List Wallets" request.

## References

- Router to extend: `src/interfaces/http/routes/wallet/routes.ts`.
- Wiring: `src/main/wallet/wallet-module.ts`, `src/infra/factories/wallet-factory.ts`.
- Existing read endpoint to mirror: `search-remittances.controller.ts`.

## Out of scope

- Pagination and filters (at most four wallets per account today).
- Showing names anywhere else (Q9).

## TDD log

### RED (Codex)

- Date:
- Tests added/changed:
- Command and failing output (summary):

### GREEN (Claude)

- Date:
- Files changed:
- Results of `npm test`, `test:db`, lint and format check:

### VERIFY (Codex)

- Date:
- Result (pass / findings):
