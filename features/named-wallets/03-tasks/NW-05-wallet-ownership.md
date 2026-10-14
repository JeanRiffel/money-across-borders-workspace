# NW-05: Only the account owner can open a wallet

| | |
|---|---|
| **Spec** | [SPEC-T1](../02-specs/specifications.md#spec-t1-wallet-ownership-authorization) |
| **Repo** | `money-across-borders` |
| **Depends on** | NW-03 (same controller and use case) |
| **Decisions** | Q6 |
| **Status** | Not started |

## Goal

Close discovery risk R3: today any authenticated user can open (and now name) a wallet on any
account. Per Q6, only the account's owner may.

**Breaking change:** the "open a wallet for a recipient" flow stops working. See the release note
in [README](README.md#release).

## How ownership is known

The login token already carries the caller's account: `LoginUseCase` signs
`{ userId, accountId }` (`src/application/user/uses-cases/login-use-case.ts:47-50`), and
`authMiddleware` puts the payload on `req.user` (`src/interfaces/http/middlewares/auth.middleware.ts:19`).
Each user has one account (`AccountRepository.findByUserId`). So "the caller owns the account"
means `body.accountId === req.user.accountId`.

## Acceptance criteria

1. `POST /wallets` with the caller's own `accountId` behaves exactly as before (201, 400, 409,
   422 cases unchanged).
2. `POST /wallets` with any other `accountId`, existing or not, returns **404** with a generic
   message (e.g. "Account not found"), so it doesn't reveal whether the account exists. No wallet
   is created.
3. A token without an `accountId` claim returns 401 or 404 (RED picks and asserts one), never 201.
4. A request rejected by criterion 2 doesn't leave an idempotency record that a later, valid
   request with the same key would replay.
5. The rule is written in `docs/invariants.md` (or an ADR): "a caller may only act on the account
   in its token", applying to `/wallets` now and meant for other routes later.
6. Swagger documents the 404 on `POST /wallets`. The Bruno "Open Wallet (Recipient)" request is
   changed to log in as the recipient, or removed.

## Notes for RED

- Tests and Cucumber features that open a recipient's wallet with the sender's identity must be
  updated to use the recipient's identity (check `features/` and the remittance use-case tests).

## Out of scope

- Authorization on `/remittances` and `/kyc`.
- The frontend part of T1 (removing the recipient flow from the UI) is in NW-10.

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
