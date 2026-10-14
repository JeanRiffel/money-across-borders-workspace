# NW-10: Name field in the Open Wallet form, own account only

| | |
|---|---|
| **Spec** | [SPEC-F2](../02-specs/specifications.md#spec-f2-wallet-name-in-the-open-wallet-form-frontend), frontend part of [SPEC-T1](../02-specs/specifications.md#spec-t1-wallet-ownership-authorization) |
| **Repo** | `money-across-borders-web` |
| **Depends on** | NW-09 (runner); NW-03 + NW-05 (backend contract) |
| **Decisions** | Q1, Q2, Q6 |
| **Status** | Not started |

## Goal

Collect the wallet name in the UI, and match the new backend rule that a user can only open
wallets on their own account. Ships together with the backend (see [README](README.md#release)).

## Acceptance criteria

Name:

1. The form has a required **Name** field (label, input, inline error), using the existing field
   pattern.
2. `lib/schemas/wallet.ts` applies the same rules as `WalletName` (NW-02): trim and collapse
   whitespace, then 1–40 grapheme clusters (`Intl.Segmenter`), no control characters. Cover the
   NW-02 cases in the schema tests (empty, blank, 40 vs 41 graphemes, `"👨‍👩‍👧"` = 1, control
   character rejected).
3. An invalid name shows an inline error, and no request is sent.
4. A valid name is sent to `/api/wallets`, and the success card shows the **name returned by the
   backend**. `OpenWalletOutput` includes `name: string`.
5. A backend 400 or 409 (e.g. duplicate name) shows the backend's message in the error toast.

Own account only (Q6):

6. The **Account ID field is removed** from the form. The route handler `app/api/wallets/route.ts`
   sets `accountId` from the session on the server and ignores any `accountId` the client sends.
7. The page copy no longer mentions opening a wallet for a recipient
   (`app/(protected)/wallets/new/page.tsx:10-12`).

General:

8. The NW-09 form tests are updated for the new fields. `npm test`, `npm run lint` and
   `npm run build` pass.
9. Checked manually against a backend with NW-03 to NW-05: create a wallet, see its name, try a
   duplicate name.

## References

- Schema: `lib/schemas/wallet.ts`. Form: `components/forms/open-wallet-form.tsx` (`defaultValues`
  line 34, fields from line 93, success card from line 57). Session: `lib/session.ts`
  (`SessionAccount.accountId`), `lib/dal.ts`.
- Read `node_modules/next/dist/docs/` before changing route handlers (repo `AGENTS.md`).

## Out of scope

- The wallet list (NW-11).

## TDD log

### RED (Codex)

- Date:
- Tests added/changed:
- Command and failing output (summary):

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
