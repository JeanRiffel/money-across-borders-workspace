# NW-11: Wallet list page

| | |
|---|---|
| **Spec** | [SPEC-F3](../02-specs/specifications.md#spec-f3-list-wallets-with-names) (frontend part) |
| **Repo** | `money-across-borders-web` |
| **Depends on** | NW-06 (backend), NW-09 (runner) |
| **Decisions** | Q5, Q9 |
| **Status** | Not started |

## Goal

Show customers their wallets and names at any time, completing "identify and distinguish their
wallets".

## Acceptance criteria

1. A new page at **`/wallets`** lists the signed-in user's wallets from `GET /wallets`. *(The spec
   allowed a page or a dashboard section; a page was chosen while writing the task. Change it here
   before RED if you prefer the dashboard.)*
2. Each row shows name, currency, balance (formatted with `lib/money.ts`) and status, in the order
   the backend returns.
3. With no wallets, an empty state links to `/wallets/new`.
4. A loading state while fetching, and an error state if the request fails.
5. The dashboard and navigation link to `/wallets`. The success card after opening a wallet links
   to it too.
6. The copy that says there is no wallet lookup is replaced: `app/(protected)/dashboard/page.tsx:43`
   and `components/forms/open-wallet-form.tsx:63`.
7. Tests cover the list, empty and error states (backend call mocked). `npm test`, `npm run lint`
   and `npm run build` pass.

## References

- Follow the existing data-fetching pattern of the remittances pages (`app/(protected)/remittances/`)
  and `lib/api`.

## Out of scope

- Renaming (SPEC-F4, deferred).

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
