# Named Wallets: Implementation Tasks

- **Source:** [specifications.md](../02-specs/specifications.md), approved 2026-10-14, with the
  answers to Q1–Q10 and Q12 recorded in its [Open Questions](../02-specs/specifications.md#open-questions).
- **Scope (v1, per Q5):** SPEC-F1, F2, F3 and SPEC-T1–T5. SPEC-F4 (rename) is deferred.
- **Method:** each task goes RED (Codex) → GREEN (Claude) → REFACTOR (Claude, optional) → VERIFY
  (Codex), as defined in the [TDD workflow](../../../docs/workflows/tdd.md). Each agent fills in its
  part of the task's `## TDD log`.

## Tasks

| ID | Task | Spec | Repo | Depends on | RED? | Status |
|---|---|---|---|---|---|---|
| [NW-01](NW-01-postgres-wallet-repository-tests.md) | Postgres wallet repository integration tests (baseline) | T4 | Backend | – | Characterization | Done |
| [NW-02](NW-02-wallet-name-domain.md) | `WalletName` value object | F1 | Backend | – | Yes | Done |
| [NW-03](NW-03-open-wallet-name-contract.md) | `Wallet` carries a name; `POST /wallets` accepts and returns it | F1 | Backend | NW-02 | Yes | Not started |
| [NW-04](NW-04-wallet-name-persistence.md) | Migration 006, database constraints and seed writers | F1, T4 | Backend | NW-01, NW-03 | Yes | Not started |
| [NW-05](NW-05-wallet-ownership.md) | Only the account owner can open a wallet | T1 | Backend | NW-03 | Yes | Not started |
| [NW-06](NW-06-list-wallets-endpoint.md) | `GET /wallets` | F3 | Backend | NW-04, NW-05 | Yes | Not started |
| [NW-07](NW-07-metrics-route-label.md) | Metrics route-label normalization | T2 | Backend | – | Yes | Not started |
| [NW-08](NW-08-idempotency-fingerprint.md) | Reject a reused Idempotency-Key with a different body | T3 | Backend | – | Yes | Not started |
| [NW-09](NW-09-frontend-test-harness.md) | Frontend test harness | T5 | Frontend | – | **No** (tooling) | Not started |
| [NW-10](NW-10-open-wallet-form-name.md) | Name field in the Open Wallet form, own account only | F2, T1 | Frontend | NW-03, NW-05, NW-09 | Yes | Not started |
| [NW-11](NW-11-wallet-list-page.md) | Wallet list page | F3 | Frontend | NW-06, NW-09 | Yes | Not started |

Status values: *Not started* → *RED* → *GREEN* → *REFACTOR* → *VERIFY* → *Done* (or back to an earlier
phase; see [TDD workflow → Failures](../../../docs/workflows/tdd.md#6-failures-and-retries)).

## Order

```
Backend:   NW-02 ─► NW-03 ─┬─► NW-04 (also needs NW-01) ─┐
                           └─► NW-05 ──────────────────────┴─► NW-06
           NW-01, NW-07, NW-08: independent, any time

Frontend:  NW-09 ─► NW-10 (needs NW-03 + NW-05) ─► NW-11 (needs NW-06)
```

## Release

NW-03, NW-04 and NW-05 change the `POST /wallets` contract in a **breaking** way: `name` becomes
required (Q1), and only the caller's own account is accepted (Q6). A frontend without NW-10, the
Bruno "Open Wallet (Recipient)" request and any other client will start getting 400 or 404.

Release them together: **migration 006 → backend (NW-03 to NW-06) → frontend (NW-10, NW-11)**, with
as little time as possible between backend and frontend.

## Decisions every task relies on

Summarized from the Open Questions; the specification is the source of truth.

| Topic | Rule |
|---|---|
| Required (Q1) | Every new wallet needs a name. |
| Validation (Q2) | Normalize first: trim the ends, collapse each run of internal whitespace to one space. Then 1–40 **grapheme clusters** (`Intl.Segmenter`, granularity `grapheme`). Any Unicode is allowed except control characters (category `Cc`); format characters such as the zero-width joiner stay allowed so emoji sequences work. |
| Existing wallets (Q3, Q8) | Migration backfills `"<CURRENCY> Wallet"` (e.g. `"USD Wallet"`), treasury wallets included; then `name` is `NOT NULL`. |
| Uniqueness (Q4) | Unique per account, case-insensitive, active and closed wallets alike. |
| Who (Q6) | Only the account's owner. The JWT already carries `accountId` (`src/application/user/uses-cases/login-use-case.ts:47-50`). Any other account → 404. |
| Shown where (Q9) | Creation confirmation and the wallet list only. |
| Idempotency (Q10) | Same key, different body → 422, use case not run. |
| Domain (Q12) | A `WalletName` value object owns validation and normalization. |
