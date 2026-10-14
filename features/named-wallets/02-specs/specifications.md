# Named Wallets: Proposed Implementation Specifications

- **Status:** Proposed. Awaiting engineering approval.
- **Date:** 2026-10-13
- **Source:** [report.md](../01-discovery/report.md) (Technical Discovery Report)
- **Brief:** [brief.md](brief.md)
- **Repositories:** `money-across-borders` (backend), `money-across-borders-web` (frontend)

This document proposes how to split the work into specifications. It does **not** contain
implementation tasks, and no source code was changed.

The open questions from the discovery are **still open**. They are collected in
[Open Questions](#open-questions), and each specification lists the questions it depends on.
Where an acceptance criterion depends on an unanswered question, it refers to the decision
(e.g. "per Q2") instead of assuming an answer.

---

## Overview

### Feature requirements

These deliver the business objective: customers can identify and distinguish their wallets by
name.

| ID | Specification | Repos | Complexity | Blocking questions |
|---|---|---|---|---|
| [SPEC-F1](#spec-f1-wallet-name-on-creation-backend--database) | Wallet name on creation (backend + database) | Backend | Medium | Q1, Q2, Q3, Q4, Q8, Q12 |
| [SPEC-F2](#spec-f2-wallet-name-in-the-open-wallet-form-frontend) | Wallet name in the Open Wallet form (frontend) | Frontend | Low | Q1, Q2 |
| [SPEC-F3](#spec-f3-list-wallets-with-names) | List wallets with names *(conditional on Q5)* | Backend + Frontend | Medium | Q5, Q6, Q9 |
| [SPEC-F4](#spec-f4-rename-a-wallet) | Rename a wallet *(conditional on Q5)* | Backend + Frontend | Medium | Q4, Q5, Q6, Q11 |

### Technical improvements found during the discovery

These are not part of the feature request. Some are **prerequisites** for the conditional feature
specs; the others are independent improvements.

| ID | Specification | Repos | Complexity | Relationship to the feature |
|---|---|---|---|---|
| [SPEC-T1](#spec-t1-wallet-ownership-authorization) | Wallet ownership authorization | Backend (+ Frontend copy) | Medium | **Prerequisite** for F3 and F4 |
| [SPEC-T2](#spec-t2-metrics-route-label-normalization) | Metrics route-label normalization | Backend | Low | **Prerequisite** for F4 (and F3 if it uses a path param) |
| [SPEC-T3](#spec-t3-idempotency-request-fingerprint-validation) | Idempotency request fingerprint validation | Backend | Medium | Independent (answers Q10) |
| [SPEC-T4](#spec-t4-postgres-wallet-repository-integration-tests) | Postgres wallet repository integration tests | Backend | Medium | Recommended alongside F1 |
| [SPEC-T5](#spec-t5-frontend-test-harness) | Frontend test harness | Frontend | Medium | Independent |

### Proposed order

```
                 ┌──────────── SPEC-T4 (recommended alongside)
                 │
 SPEC-F1 ────────┼──► SPEC-F2                     ← minimum viable feature
   │             │
   │   SPEC-T1 ──┼──► SPEC-F3 ──► SPEC-F4         ← only if Q5 includes list / rename
   │             │                  ▲
   └─────────────┘     SPEC-T2 ─────┘

 SPEC-T3, SPEC-T5: independent, can be scheduled at any time
```

**Minimum viable delivery:** SPEC-F1 + SPEC-F2. This meets the brief's literal requirement
("assign a name"). Customers would only see the name on the confirmation card after creating the
wallet (discovery risk R2).

---

## Feature Requirements

### SPEC-F1: Wallet name on creation (backend + database)

**Business objective**
Let customers give a wallet a name when they create it, so it can later be identified by
something more meaningful than its ID or currency.

**Functional scope**
- In scope:
  - New `name` column on the `wallets` table, added by a new idempotent migration
    (`006_add_wallet_name.sql`), compatible with existing rows.
  - `Wallet` domain entity holds the name and keeps it through every new instance it creates
    (`credit()`, `debit()`).
  - `POST /wallets` accepts `name` in the request body and returns it in the response
    (`OpenWalletInput`, `OpenWalletOutput`, `OpenWalletUseCase`).
  - `PostgresWalletRepository` reads and writes the name; the in-memory treasury seed is updated
    for the new constructor.
  - Swagger schemas and the Bruno `Open Wallet` requests are updated.
- Out of scope:
  - Listing or renaming wallets (SPEC-F3, SPEC-F4).
  - Frontend changes (SPEC-F2).
  - Adding the name to remittances, events or the Elasticsearch index (Q9).

**Acceptance criteria**
1. `POST /wallets` with a valid `name` returns 201, and `result.name` equals the stored value
   (after any normalization decided in Q2).
2. `POST /wallets` without `name`:
   - if the name is optional (Q1), returns 201 with the agreed "no name" value;
   - if the name is required (Q1), returns 400 with a validation message.
3. A `name` that breaks the agreed rules (Q2: length, characters, whitespace) returns 400 and
   creates no wallet.
4. If names must be unique (Q4), a duplicate name for the same account is rejected with the
   agreed status code, including when two requests race (database-level enforcement).
5. After a remittance debits or credits the wallet, the wallet still has the same name, in both
   the in-memory and Postgres repositories.
6. Running the migration twice in a row succeeds (idempotent), and existing wallets, including
   the treasury wallets, are handled as decided in Q3 and Q8.
7. Existing clients that don't send `name` keep working, unless Q1 makes it required. In that
   case the change is documented as breaking, and the release order is migration → backend →
   frontend.
8. Swagger shows `name` in `OpenWalletInput` and `OpenWalletOutput`, and the Bruno requests
   include an example name.
9. Unit tests cover:
   - entity pass-through in `credit`/`debit`;
   - DTO validation cases;
   - use-case output.
   Existing wallet and remittance tests still pass.

**Dependencies**
- Decisions: Q1, Q2, Q3, Q4, Q8, Q12.
- Technical: none. SPEC-T4 is recommended so the new SQL has automated coverage.

**Affected repositories:** `money-across-borders`

**Estimated complexity: Medium.** The change touches every layer (migration, domain,
application, infrastructure, interface, docs) but follows existing patterns. It would go up if
Q4 requires case-insensitive uniqueness, or if Q3 requires a backfill.

---

### SPEC-F2: Wallet name in the Open Wallet form (frontend)

**Business objective**
Let customers enter a wallet name in the UI when they open a wallet, and see it confirmed.

**Functional scope**
- In scope:
  - `lib/schemas/wallet.ts`: `openWalletSchema` and the `OpenWalletOutput` type include `name`,
    with the same rules as the backend.
  - `components/forms/open-wallet-form.tsx`: new Name field (label, input, error message,
    default value); the success card shows the name.
  - Copy on `app/(protected)/wallets/new/page.tsx` if needed.
- Out of scope:
  - Wallet list or rename UI (SPEC-F3, SPEC-F4).
  - Remittance screens.

**Acceptance criteria**
1. The Open Wallet form shows a Name field, marked required or optional per Q1.
2. Client-side validation matches the backend rules (Q2). Invalid input shows an inline field
   error, and no request is sent.
3. Submitting a valid name creates the wallet, and the success card shows the name returned by
   the backend.
4. If the name is optional and left empty, the success card shows the agreed fallback (Q3).
5. A backend validation or duplicate-name error (SPEC-F1 criteria 3 and 4) appears as an error
   toast with the backend's message.
6. The `/api/wallets` route forwards `name` to the backend. The name isn't stripped by schema
   drift (discovery risk R5).
7. `npm run lint` and `npm run build` pass. The flow is checked manually against a backend that
   includes SPEC-F1.

**Dependencies**
- Specs: **SPEC-F1** (backend contract must be deployed first).
- Decisions: Q1, Q2 (and Q3 for the empty-name fallback).

**Affected repositories:** `money-across-borders-web`

**Estimated complexity: Low.** One schema, one form component, and no new pages. There's no
automated frontend test coverage (see SPEC-T5).

---

### SPEC-F3: List wallets with names

> **Conditional:** only if Q5 includes listing wallets in v1.

**Business objective**
Let customers see their wallets and names at any time, not only right after creation. This fully
meets the goal of being able to "identify and distinguish their wallets".

**Functional scope**
- In scope:
  - **Backend:**
    - new `GET /wallets` endpoint returning the authenticated customer's wallets (id, name,
      currency, balance, status, createdAt);
    - new `WalletRepository.findByAccountId` method, in Postgres and in-memory;
    - new use case, DTO, controller and route; Swagger and Bruno updates.
  - **Frontend:**
    - new wallet list (a page, or a section on the dashboard);
    - a Next route handler that calls the backend;
    - an "unnamed wallet" fallback (Q3);
    - updated copy where the UI currently says there's no wallet lookup
      (`dashboard/page.tsx:43`, `open-wallet-form.tsx:63`).
- Out of scope:
  - Renaming (SPEC-F4).
  - Pagination or filtering, unless product asks for it. The current rules allow at most one
    wallet per supported currency per account.

**Acceptance criteria**
1. `GET /wallets` returns only wallets for accounts the caller is allowed to see (SPEC-T1 / Q6),
   including their names.
2. Unauthenticated requests return 401. Requests for an account the caller isn't allowed to see
   are rejected with the status agreed in SPEC-T1.
3. Wallets without a name are returned with the agreed "no name" value. The UI shows the agreed
   fallback (Q3).
4. Treasury wallets are never returned to customers.
5. The frontend shows the list with name, currency, balance and status, and handles empty,
   loading and error states.
6. If Q9 extends the name to other screens, those screens are listed in this spec before
   approval.
7. Backend unit tests cover the use case and both repository implementations. Swagger and Bruno
   are updated.

**Dependencies**
- Specs: **SPEC-F1**, **SPEC-T1** (ownership rule must exist before exposing wallet data).
  **SPEC-T2** too, if the endpoint design uses a path parameter (e.g.
  `/accounts/:id/wallets`).
- Decisions: Q5, Q6, Q9 (and Q3 for the fallback).

**Affected repositories:** `money-across-borders`, `money-across-borders-web`

**Estimated complexity: Medium.** The first read endpoint in the wallet context, plus a new
frontend view.

---

### SPEC-F4: Rename a wallet

> **Conditional:** only if Q5 includes renaming in v1.

**Business objective**
Let customers change a wallet's name after creation, so names stay meaningful as their use of
the wallet changes.

**Functional scope**
- In scope:
  - **Backend:**
    - new `PATCH /wallets/:id` (or an equivalent agreed design) that updates only the name;
    - new `Wallet.rename()` method that returns a new instance, in line with the entity's
      immutable style;
    - `PostgresWalletRepository.save()` updates `name` on conflict;
    - use case, DTO, controller, route; Swagger and Bruno.
  - **Frontend:** a rename control on the wallet list (SPEC-F3), with a route handler.
  - **Audit**, if Q11 requires it: a `wallet.renamed` event through the transactional outbox.
- Out of scope:
  - Changing any other wallet attribute.
  - Renaming treasury wallets.

**Acceptance criteria**
1. The wallet's owner can rename it. The new name is returned and persisted, and the balance,
   status and currency are unchanged.
2. Renaming a wallet the caller doesn't own is rejected (SPEC-T1).
3. The new name follows the same validation (Q2) and uniqueness (Q4) rules as creation.
4. Renaming a wallet that doesn't exist returns 404. Treasury wallets can't be renamed.
5. A rename that races with a remittance on the same wallet loses neither the new name nor the
   balance change.
6. Retrying with the same Idempotency-Key doesn't apply the rename twice, and replays the result
   in line with the existing `IdempotentDecorator` behavior (or SPEC-T3, if delivered).
7. If Q11 requires audit, each successful rename writes exactly one outbox event in the same
   transaction.
8. HTTP metrics for this route use a bounded label (SPEC-T2).
9. The frontend shows the updated name without a manual page reload, and shows validation and
   permission errors.

**Dependencies**
- Specs: **SPEC-F1**, **SPEC-F3** (UI host), **SPEC-T1** (ownership), **SPEC-T2** (path-param
  route).
- Decisions: Q4, Q5, Q6, Q11.

**Affected repositories:** `money-across-borders`, `money-across-borders-web`

**Estimated complexity: Medium.** It becomes **High** if Q11 requires an audit event, because
that adds outbox, relay and consumer work.

---

## Technical Improvements Discovered During the Investigation

### SPEC-T1: Wallet ownership authorization

**Business objective**
Make sure customers can only create, view or change wallets on accounts they own. This protects
customer data and stops people from naming other customers' wallets.

**Background:** discovery risk R3. `authMiddleware` only checks the JWT. `OpenWalletController`
accepts any `accountId`. The current UI explicitly lets users open wallets on a recipient's
account (`app/(protected)/wallets/new/page.tsx:11`).

**Functional scope**
- In scope:
  - A rule that links the JWT subject to the account(s) it may act on, applied to `/wallets`
    endpoints. This covers new endpoints in F3/F4, and `POST /wallets` if Q6 says so.
  - Matching frontend copy changes, if opening wallets for recipients is no longer allowed.
- Out of scope: authorization for `/remittances` and `/kyc` (can follow the same pattern
  later).

**Acceptance criteria**
1. The agreed ownership rule (Q6) is written down in `docs/invariants.md` or an ADR.
2. Requests on an account the caller doesn't own are rejected with the agreed status
   (e.g. 403 or 404), and no data is leaked.
3. If Q6 keeps "open a wallet for a recipient", that path still works, and who may set its name
   is defined.
4. Unit tests cover both allowed and denied cases.

**Dependencies**
- Decisions: Q6.
- It is a **prerequisite** for SPEC-F3 and SPEC-F4.

**Affected repositories:** `money-across-borders`; `money-across-borders-web` (copy/flow only,
if the recipient flow changes)

**Estimated complexity: Medium.** It's a new cross-cutting rule. It's also a behavior change for
the existing `POST /wallets` if applied there.

---

### SPEC-T2: Metrics route-label normalization

**Business objective**
Keep monitoring reliable and cheap as the API adds routes with path parameters.

**Background:** discovery risk R9. `src/infra/observability/metrics.ts:72-82` uses `req.path` as
the Prometheus `route` label, and assumes all routes are static. A route like `/wallets/:id`
would create one label per wallet ID.

**Functional scope**
- In scope: use the matched route template (e.g. `/wallets/:id`) instead of the raw path, with a
  bounded fallback for unmatched requests.
- Out of scope: other metric changes.

**Acceptance criteria**
1. Requests to a parameterized route produce a single `route` label value, the template, no
   matter which IDs are used.
2. Static routes keep their current label values, so existing dashboards and alerts don't break.
3. Unmatched routes (404s) map to one fixed label, not the raw path.
4. A unit test checks the label for static, parameterized and unmatched routes.

**Dependencies**
- None.
- It is a **prerequisite** for SPEC-F4, and for SPEC-F3 if it uses a path parameter.

**Affected repositories:** `money-across-borders`

**Estimated complexity: Low.**

---

### SPEC-T3: Idempotency request fingerprint validation

**Business objective**
Stop clients from getting a misleading success when they reuse an Idempotency-Key with a
different request.

**Background:** discovery risk R4. `IdempotentDecorator`
(`src/application/shared/idempotency/idempotent-decorator.ts`) replays the cached response for a
key without comparing request bodies. For example, a retry with a different wallet name returns
the original wallet. `idempotency_records.request_hash` exists in `001_init_schema.sql` but is
never used. Production uses the Redis repository.

**Functional scope**
- In scope:
  - Store a fingerprint of the request alongside each idempotency record.
  - Reject a reused key whose request doesn't match.
  - Apply this to every endpoint wrapped by `IdempotentDecorator` (accounts, wallets,
    remittances), in both the Redis and in-memory repositories (and Postgres, if it's kept).
- Out of scope: changing the in-flight/claim semantics.

**Acceptance criteria**
1. The same key with the same body replays the original response, as today.
2. The same key with a different body is rejected with the agreed error (e.g. 422 or 409) and a
   clear message, and doesn't execute the use case.
3. The fingerprint ignores differences that don't change the meaning of the request (e.g. key
   order), and the chosen normalization is documented.
4. Existing idempotency tests (`idempotent-decorator.test.ts`, the
   `send-remittance-idempotency.test.ts` suite, the repository tests) still pass, plus new
   mismatch tests.
5. ADR 0003 (Redis-backed idempotency) is updated.

**Dependencies**
- Decisions: Q10 (whether the current replay behavior is acceptable; if it is, this spec can be
  deferred).
- Not a prerequisite for any feature spec.

**Affected repositories:** `money-across-borders`

**Estimated complexity: Medium.** It changes the behavior of every write endpoint.

---

### SPEC-T4: Postgres wallet repository integration tests

**Business objective**
Catch SQL mistakes in wallet persistence before release, not in production.

**Background:** discovery risk R7. `npm test` runs only against in-memory repositories
(`docs/known-issues.md:9`). `PostgresWalletRepository` (its upsert and SELECTs) has no automated
test, and SPEC-F1 changes all of its queries.

**Functional scope**
- In scope: integration tests that run `PostgresWalletRepository` against a real Postgres. They
  reuse the existing `__tests__/concurrency/support/db.ts` setup if it fits, and cover save
  (insert and upsert), `findById` and `findByAccountIdAndCurrency`.
- Out of scope: integration tests for other repositories.

**Acceptance criteria**
1. The tests run against a migrated Postgres instance using a documented command (and in CI, if
   the CI pipeline can provide Postgres).
2. They check that every column, including `name` once SPEC-F1 lands, round-trips through
   save → find.
3. They check that the upsert updates the intended columns and keeps `id`, `account_id`,
   `currency` and `created_at`.
4. They check that the `(account_id, currency)` uniqueness, and the name uniqueness if Q4
   requires it, are enforced by the database.

**Dependencies**
- None to build. Most valuable when delivered together with SPEC-F1.

**Affected repositories:** `money-across-borders`

**Estimated complexity: Medium.** Mostly test infrastructure and CI setup.

---

### SPEC-T5: Frontend test harness

**Business objective**
Give the frontend automated checks, so form and API-contract changes like SPEC-F2 can be
verified without manual testing.

**Background:** `money-across-borders-web` has no test files and no `test` script
(`package.json` has only `build` and `lint`).

**Functional scope**
- In scope:
  - A test runner suited to the project's Next.js version (see that repo's `AGENTS.md` warning
    about breaking changes).
  - A `test` script.
  - First tests for the zod schemas in `lib/schemas/` and the Open Wallet form.
- Out of scope: end-to-end browser tests (could be a later spec).

**Acceptance criteria**
1. `npm test` runs and passes locally.
2. Tests cover `openWalletSchema` valid and invalid cases, and the Open Wallet form's
   submit/validation behavior.
3. The repository docs explain how to run the tests.

**Dependencies:** none. If delivered before SPEC-F2, SPEC-F2 should add tests for the name
field.

**Affected repositories:** `money-across-borders-web`

**Estimated complexity: Medium.** New tooling in a Next.js version with breaking changes.

---

## Open Questions

Copied from the discovery report (section 8) and still **unanswered**. They need product or
engineering decisions before the specifications that depend on them are approved for
implementation.

| ID | Question | Blocks |
|---|---|---|
| Q1 | **Required or optional?** Must every new wallet have a name, or is it optional? | F1, F2 |
| Q2 | **Validation rules:** min/max length? Allowed characters (emoji, unicode, punctuation)? Trim whitespace? Collapse internal whitespace? | F1, F2, F4 |
| Q3 | **Existing wallets:** leave unnamed (NULL plus a UI fallback), or backfill a generated default? If backfilling, what format? | F1, F2, F3 |
| Q4 | **Uniqueness:** must names be unique per account? Case-sensitive or not? Across active and closed wallets? | F1, F4, T4 |
| Q5 | **Scope of v1:** set the name only at creation, rename later (needs `PATCH`), or list wallets with names (needs `GET`)? Without the list, the name is visible only once. | F3, F4 (whether they're in scope) |
| Q6 | **Authorization:** who may name a wallet: only the owning account's user, or anyone opening it (the current UI lets users open wallets for recipients)? Should this feature introduce the missing ownership check on `/wallets`? | T1, F3, F4 |
| Q7 | **Product value under the current constraint:** with one wallet per currency per account, is the goal mainly a friendly label? Or is the `UNIQUE(account_id, currency)` rule expected to be relaxed later, so that one account can hold several wallets in the same currency? That would be a much larger change, outside these specs: `findByAccountIdAndCurrency` drives remittances. | All (confirms the overall approach) |
| Q8 | **Treasury wallets:** should system treasury wallets get names (e.g. `"Treasury USD"`), or stay NULL? | F1 |
| Q9 | **Where else is the name shown?** Remittance form/history, the Elasticsearch remittance index, events (`remittance.completed`)? | F3 (and possibly new specs) |
| Q10 | **Idempotency semantics:** is replaying the original response for the same key with a different name acceptable? | T3 |
| Q11 | **Audit:** do renames need history (e.g. a `wallet.renamed` outbox event)? | F4 |
| Q12 | **Domain modelling:** a `WalletName` value object, or DTO-level validation only? | F1 |

**Decisions needed first:** Q5 and Q7 settle which specs are in scope at all, and Q1–Q4 settle
the core rules. Answering those six would unblock the minimum viable delivery (SPEC-F1 +
SPEC-F2).

---

## Requested Approval

Engineering is asked to:

1. **Approve, change or reject** each specification (F1–F4, T1–T5).
2. **Assign an owner** to each open question, Q5 and Q7 first.
3. **Confirm the release order:** migration → backend → frontend.
4. **Decide** whether the technical improvements T3–T5 are scheduled with this feature or added
   to the backlog.

Implementation tasks will be created only after approval.

| Spec | Decision (Approve / Change / Reject / Defer) | Notes |
|---|---|---|
| SPEC-F1 | | |
| SPEC-F2 | | |
| SPEC-F3 | | |
| SPEC-F4 | | |
| SPEC-T1 | | |
| SPEC-T2 | | |
| SPEC-T3 | | |
| SPEC-T4 | | |
| SPEC-T5 | | |
