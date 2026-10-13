# Technical Discovery Report: Named Wallets

- **Brief:** [brief.md](brief.md)
- **Date:** 2026-10-13
- **Repositories inspected:** `money-across-borders` (backend), `money-across-borders-web` (frontend)
- **Changes made:** none. No source, database or migration changes; this is a read-only investigation.

---

## Summary

- Wallets **have no name field today**: not in the domain entity, DTOs, API contract, Postgres
  schema, seed tooling or frontend.
- Wallets are created through one flow: `POST /wallets` → `OpenWalletUseCase`. It is the only
  wallet endpoint. **No endpoint lists, reads or updates wallets.**
- That last point limits the feature the most. The backend can store a name, but the frontend
  can only show it once, on the confirmation card right after the wallet is created. Showing it
  anywhere else, or letting customers rename later, needs new read and update endpoints that
  don't exist yet.
- A database migration **is required**: a new `wallets.name` column. If it is nullable, or has a
  default, it is additive and works with existing rows, following the same pattern as
  `004_add_wallet_version.sql`.
- Wallets are already unique per `(account_id, currency)`, so each account has at most one wallet
  per currency (4 supported currencies: USD, BRL, EUR, GBP). Product needs to confirm what a name
  adds, given that constraint (see Open Questions).

---

## 1. Current Behavior

### Backend: wallet creation flow

```
POST /wallets  (JWT required; Idempotency-Key header optional)
  └─ walletRouter                      src/interfaces/http/routes/wallet/routes.ts:58
     └─ OpenWalletController.handle    src/interfaces/http/controllers/open-wallet.controller.ts:31
        ├─ OpenWalletInput.from(req.body)   zod validation → 400 on failure
        └─ IdempotentDecorator.execute      src/application/shared/idempotency/idempotent-decorator.ts:11
           └─ OpenWalletUseCase.execute     src/application/wallet/uses-cases/open-wallet-use-case.ts:20
              ├─ walletRepository.findByAccountIdAndCurrency → WalletAlreadyExistsError (409)
              ├─ new Wallet(id, accountId, currency, balance, ACTIVE, now)
              ├─ walletRepository.save(wallet)  (Postgres upsert)
              └─ OpenWalletOutput.from(wallet)
```

Wiring: `src/main/wallet/wallet-module.ts` (`buildWalletModule`) and
`src/infra/factories/wallet-factory.ts` (`createOpenWalletUseCase`). The router is mounted in
`src/main/server.ts:154`.

**Request body** (`OpenWalletInput`, `src/application/wallet/dto/open-wallet-input.ts:7`):
`accountId` (uuid), `currency` (3 chars), `initialBalanceMinorUnits` (optional int ≥ 0).
The zod schema is a plain `z.object`, so **unknown keys are silently dropped**: a `name` sent
today is accepted and then ignored.

**Response** (`OpenWalletOutput`, `src/application/wallet/dto/open-wallet-output.ts`):
`walletId, accountId, currency, balanceMinorUnits, status, createdAt`, wrapped in the
`{ statusCode, result }` envelope.

### Backend: Wallet domain entity

`src/domain/wallet/entities/wallet.ts:8`: an immutable-style class whose constructor takes 6
positional arguments: `id, accountId, currency, balance, status, createdAt`. `credit()` (line 44)
and `debit()` (line 56) **return new `Wallet` instances** and re-pass every constructor argument.

### Backend: persistence

- `WalletRepository` interface (`src/domain/wallet/repository/wallet-repository.ts`): `save`,
  `findById`, `findByAccountIdAndCurrency`. There is no list method and no update method beyond
  `save`.
- `PostgresWalletRepository` (`src/infra/persistence/postgresql/postgres-wallet-repository.ts`):
  - `WalletRow` type and `toWallet()` mapper (lines 10–29)
  - `save()` is an `INSERT … ON CONFLICT (id) DO UPDATE SET balance_minor_units, status_id`
    (lines 35–51)
  - both `SELECT`s list their columns explicitly (lines 55, 67)
- `InMemoryWalletRepository` (`src/infra/persistence/in-memory/in-memory-wallet-repository.ts`)
  stores `Wallet` objects; `npm test` uses it.

### Database: `wallets` table

Defined in `src/infra/persistence/postgresql/migrations/001_init_schema.sql:49`, then changed by
`004_add_wallet_version.sql`:

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | app-generated uuidv7 |
| `account_id` | UUID NOT NULL FK → accounts | |
| `currency` | CHAR(3) NOT NULL | |
| `balance_minor_units` | BIGINT NOT NULL, CHECK ≥ 0 | |
| `status_id` | SMALLINT NOT NULL | 1=ACTIVE, 2=CLOSED |
| `created_at` | TIMESTAMPTZ NOT NULL DEFAULT now() | |
| `version` | INTEGER NOT NULL DEFAULT 0 | added in 004; concurrency lab only |
| — | `UNIQUE (account_id, currency)` | one wallet per currency per account |

**There is no `name` column.** Migrations are plain `.sql` files that
`migrations/run-migrations.ts` applies in filename order. There is no migration-tracking table:
every file is re-run each time, so new migrations **must be idempotent** (`IF NOT EXISTS`).

### Other places that write wallet rows

- `migrations/002_seed_treasury_wallets.sql`: system treasury wallets (one per currency).
- `src/infra/persistence/in-memory/seed-treasury-wallets.ts:26`: in-memory equivalent
  (`new Wallet(...)`).
- `src/infra/seed/persistence/batch-writer.ts` (`insertWallets`, about line 76) and
  `src/infra/seed/types.ts:28` (`SeedWalletRow`): bulk seed tooling. It uses explicit column
  lists, so it is unaffected if the new column is nullable.
- `src/infra/persistence/postgresql/concurrency-lab/*`: touch only `id`, `balance_minor_units`
  and `version`. Unaffected.

### Remittances

`SendRemittanceUseCase` (`src/application/remittance/uses-cases/send-remittance-use-case.ts:59-72`)
finds wallets by `(accountId, currency)`, calls `debit()`/`credit()` and saves the result.
Remittance DTOs, the Elasticsearch read model and the frontend remittance table do **not** expose
wallet ids or wallet data.

### Frontend: wallet creation flow

```
/wallets/new                     app/(protected)/wallets/new/page.tsx
  └─ <OpenWalletForm>            components/forms/open-wallet-form.tsx
     ├─ react-hook-form + zodResolver(openWalletSchema)
     ├─ fields: Account ID, Currency (Select), Initial balance
     └─ fetch POST /api/wallets
        └─ Next route handler    app/api/wallets/route.ts
           ├─ requireApiSession(), parseJsonBody(openWalletSchema)
           └─ backendFetch("/wallets", { idempotencyKey: newIdempotencyKey(), body: parsed.data })
```

- `lib/schemas/wallet.ts`: `openWalletSchema` (mirrors the backend input) and the
  `OpenWalletOutput` type (mirrors the backend output).
- The success view (`open-wallet-form.tsx:57-88`) shows Wallet ID, Account ID, Currency, Balance
  and Status, and states: *"There's no lookup endpoint for this later — save these details now."*
- The dashboard (`app/(protected)/dashboard/page.tsx:43`) says the same: *"The backend has no
  account/wallet lookup endpoint."* **The frontend has no wallet list or detail view.**
- The `/wallets/new` page lets a user open a wallet for **any** account ID, including a
  recipient's (page copy, line 11). The backend doesn't check that `accountId` belongs to the JWT
  subject (`auth.middleware.ts` only verifies the token).

---

## 2. Proposed Behavior

Minimum scope that fits the brief ("customers must be able to assign a name to their wallets"):

1. `POST /wallets` accepts an optional or required `name` (see Open Questions) in the request
   body.
2. The name is validated, stored on the `Wallet` entity, and persisted in `wallets.name`.
3. `OpenWalletOutput` returns `name`.
4. The frontend "Open a wallet" form collects the name, and the success card shows it.
5. Existing wallets, which have no name, keep working unchanged.

Optional scope. These need new endpoints, so they are **decisions, not assumptions**:

6. Rename an existing wallet (e.g. `PATCH /wallets/:id`).
7. List the customer's wallets with their names (e.g. `GET /wallets`), so the name can be shown
   anywhere other than the one-time confirmation card.

Without 6 and 7, the "identify and distinguish their wallets" goal is only partly met: customers
can only see a name at the moment they create the wallet.

---

## 3. Affected Components and Files

### Backend (`money-across-borders`)

| Layer | File | Change type |
|---|---|---|
| Domain | `src/domain/wallet/entities/wallet.ts` | **Modify**: add field, getter, carry it through `credit`/`debit` |
| Domain (optional) | `src/domain/wallet/value-objects/wallet-name-value-object.ts` | **New** (if a value object is chosen) |
| Domain | `src/domain/shared/errors.ts` | Possibly: new error if domain-level validation is added |
| Application | `src/application/wallet/dto/open-wallet-input.ts` | **Modify**: zod schema + constructor |
| Application | `src/application/wallet/dto/open-wallet-output.ts` | **Modify**: add `name` |
| Application | `src/application/wallet/uses-cases/open-wallet-use-case.ts` | **Modify**: pass name to `new Wallet` |
| Infra | `src/infra/persistence/postgresql/postgres-wallet-repository.ts` | **Modify**: `WalletRow`, `toWallet`, `INSERT`, `ON CONFLICT`, both `SELECT`s |
| Infra | `src/infra/persistence/in-memory/in-memory-wallet-repository.ts` | No change expected (stores entity objects) |
| Infra | `src/infra/persistence/in-memory/seed-treasury-wallets.ts` | **Modify**: constructor call |
| Infra | `src/infra/persistence/postgresql/migrations/006_add_wallet_name.sql` | **New** migration (not created, per constraints) |
| Infra (optional) | `src/infra/seed/types.ts`, `src/infra/seed/persistence/batch-writer.ts`, `src/infra/seed/generators/wallet-generator.ts` | Only if seeded wallets should have names |
| Interface | `src/interfaces/http/docs/swagger.ts` (`OpenWalletInput` / `OpenWalletOutput` schemas, ~lines 103–128) | **Modify** |
| Interface | `src/interfaces/http/routes/wallet/routes.ts` | Only for new endpoints (rename/list) |
| Interface | `src/interfaces/http/controllers/open-wallet.controller.ts` | No change expected (validation errors already map to 400) |
| Tooling | `bruno-collection/Wallet/Open Wallet (Sender).bru`, `…(Recipient).bru` | **Modify**: add `name` to the example bodies |
| Docs | `AGENTS.md` / `docs/architecture.md` / `docs/invariants.md` | Review for wallet descriptions |

### Frontend (`money-across-borders-web`)

| File | Change type |
|---|---|
| `lib/schemas/wallet.ts` | **Modify**: `openWalletSchema` + `OpenWalletOutput` type |
| `components/forms/open-wallet-form.tsx` | **Modify**: new Name field, default value, success-card row |
| `app/api/wallets/route.ts` | No code change expected (forwards `parsed.data`), but it is part of the contract |
| `app/(protected)/wallets/new/page.tsx` | Possibly: copy update |
| `app/(protected)/dashboard/page.tsx` | Only if a wallet list is added |
| New page/component (e.g. `app/(protected)/wallets/page.tsx`) | Only if list/rename endpoints are added |

---

## 4. Required Backend Changes

1. **Entity** (`wallet.ts`)
   - Add a `name` constructor parameter and a `getName()` getter. **Append it after `createdAt`**
     (or make it optional) to keep the positional-argument churn small.
   - **Critical:** `credit()` (line 46) and `debit()` (line 61) build a new `Wallet` and must pass
     `name` through. Otherwise every remittance silently drops the name from the entity. With
     Postgres, the `ON CONFLICT` clause would still keep the stored value, but the in-memory repo
     (and every test) would lose it.
   - If renaming is in scope: add a `rename(name): Wallet` method that returns a new instance,
     consistent with the immutable style (`docs/architecture.md:106`).
   - Decide where validation lives: a `WalletName` value object (like `Currency`/`WalletId`) or
     only the zod DTO. The codebase uses value objects for domain concepts; ADR 0009 puts *shape*
     validation in zod and *business* validation in the domain.
2. **Input DTO** (`open-wallet-input.ts`)
   - Add `name` to `openWalletInputSchema`, e.g. `z.string().trim().min(1).max(N)`, as
     `.optional()` or required depending on Q1/Q2. Add a constructor field and map it in `from()`.
3. **Output DTO** (`open-wallet-output.ts`): add `name` (`string | null` if optional).
4. **Use case** (`open-wallet-use-case.ts:29`): pass `input.name` into `new Wallet(...)`. If names
   must be unique per account (Q4), add a repository lookup and a new error, and map that error
   to an HTTP status in `OpenWalletController`.
5. **Postgres repository** (`postgres-wallet-repository.ts`)
   - `WalletRow`: add `name: string | null`.
   - `toWallet()`: map `row.name`.
   - `save()`: add `name` to the column list, `VALUES` (`$7`) and `ON CONFLICT … DO UPDATE SET`
     (needed for rename; harmless otherwise).
   - `findById` / `findByAccountIdAndCurrency`: add `name` to the `SELECT` lists.
6. **Treasury seed** (`seed-treasury-wallets.ts:26`): update the constructor call (null, or a
   system name, see Q8).
7. **Swagger** (`swagger.ts`): add `name` to the `OpenWalletInput` / `OpenWalletOutput` schemas,
   and to the `required` list if mandatory.
8. **Bruno collection**: add `name` to the example request bodies.
9. **Optional, only if in scope:**
   - `GET /wallets`: needs a new `WalletRepository.findByAccountId(accountId)` method, a new use
     case/DTO/controller/route, an in-memory and Postgres implementation, and an
     account-ownership rule (no ownership check exists today).
   - `PATCH /wallets/:id`: needs a new use case, `Wallet.rename()`, an ownership check, and a
     decision on idempotency (the existing `IdempotentDecorator` pattern). Note:
     `src/infra/observability/metrics.ts:72` assumes routes are static ("no /wallets/:id-style
     params") and uses `req.path` as a metric label. A path-param route would break that
     assumption and **cause unbounded metric-label cardinality**, so it needs a change there too.

### Tests that need updating (backend)

| Test | Why |
|---|---|
| `__tests__/domain/entities/wallet.test.ts` | `buildWallet()` (line 13) calls the constructor; add a test that `credit`/`debit` keep `name` |
| `__tests__/application/use-cases/wallet/open-wallet-use-case.test.ts` | Add a test that `name` is set and returned; existing tests break if `name` becomes required |
| `__tests__/application/wallet/dto/open-wallet-input.test.ts` | Add validation cases: empty, whitespace, too long, missing (if required), wrong type |
| `__tests__/application/use-cases/remittance/send-remittance-use-case.test.ts`, `send-remittance-idempotency.test.ts` | Indirect: they open wallets; break only if `name` becomes required. Could also check that the name survives a remittance |
| `__tests__/infra/seed/unit/generate-dataset.test.ts`, `__tests__/infra/seed/integration/seed-pipeline.test.ts` | Only if seed rows change |
| `__tests__/concurrency/*` | Use raw SQL against `wallets`; unaffected if the column is nullable/defaulted |

No test exercises `PostgresWalletRepository` directly (`npm test` runs in-memory only, per
`docs/known-issues.md:9`), so the SQL changes have **no automated coverage** today.

---

## 5. Required Frontend Changes

1. **Schema** (`lib/schemas/wallet.ts`)
   - Add `name` to `openWalletSchema`, matching the backend rules exactly (trim, min, max,
     optional/required).
   - Add `name` to the `OpenWalletOutput` type.
2. **Form** (`components/forms/open-wallet-form.tsx`)
   - Add a `name` default in `defaultValues` (line 34).
   - Add a new `<Field>` with `<FieldLabel htmlFor="name">`, `<Input {...register("name")} />`
     and `<FieldError>`, following the existing fields (lines 93–133). **Where to collect it:**
     before or after Currency in the form.
   - Show the name in the success card `<dl>` (lines 67–78). This is currently the **only** place
     the name can be displayed.
3. **API route** (`app/api/wallets/route.ts`): no logic change, because it forwards
   `parsed.data`. However, `parseJsonBody` uses the same zod schema, so a frontend schema that
   doesn't include `name` would **silently strip it** before it reaches the backend.
4. **Where to display the name**
   - With the current backend: only on the creation success card.
   - If `GET /wallets` is added: a new wallet list page/section (e.g. on the dashboard, replacing
     the "no lookup endpoint" notes at `dashboard/page.tsx:43` and `open-wallet-form.tsx:63`).
   - If rename is added: an edit control on that list.
   - The remittance form and table don't refer to wallets (they use account + currency), so they
     need no change unless product wants the wallet name shown there.
5. **Frontend tests:** the repo has **no test files** and no `test` script in `package.json` (only
   `build` and `lint`). Verification would be manual or `next build` + `eslint`.
6. **Note:** `money-across-borders-web/AGENTS.md` warns that this Next.js version has breaking
   API changes, and that `node_modules/next/dist/docs/` should be read before writing code.

### API contract changes

| Contract | Before | After |
|---|---|---|
| `POST /wallets` request | `{ accountId, currency, initialBalanceMinorUnits? }` | `+ name` (optional/required TBD) |
| `POST /wallets` 201 response `result` | `{ walletId, accountId, currency, balanceMinorUnits, status, createdAt }` | `+ name` (`string` or `string \| null`) |
| `POST /wallets` 400 | zod message | new messages for name validation |
| `POST /wallets` 409 | duplicate currency / in-flight key | possibly also duplicate name (if Q4 = unique) |
| New (optional) | — | `GET /wallets`, `PATCH /wallets/:id` |

Adding an **optional** `name` is backward compatible for both repos and for existing Bruno
requests. Making it **required** is a breaking change: the backend and frontend must deploy
together (backend first would reject the old frontend's requests).

---

## 6. Database Implications

- **Migration required:** yes. Add a `name` column to `wallets`. This would be a new file,
  `006_add_wallet_name.sql`; it was **not created**, per constraints.
- **Shape to decide:** probably `TEXT` or `VARCHAR(N)`. The column must be
  - **nullable**, or `NOT NULL DEFAULT ''` / a generated default, so that existing rows stay
    valid. `ADD COLUMN … NULL` (or with a constant default) is a metadata-only change in
    Postgres 11+, with no table rewrite and only a brief lock.
  - **idempotent:** `ADD COLUMN IF NOT EXISTS`, because `run-migrations.ts` re-applies every file
    on every run.
- **Existing records:** every existing wallet, including the 4 treasury wallets from migration
  002, will have no name. Options (Q3):
  - leave them NULL and let the UI show a fallback (e.g. the currency code);
  - backfill a generated name (e.g. `'USD wallet'`) in the migration, which would allow
    `NOT NULL` afterwards.
- **Optional constraints** (depending on business rules):
  - `CHECK (char_length(name) BETWEEN 1 AND N)` for defense in depth;
  - `UNIQUE (account_id, name)` if names must be unique per account. With nullable names, Postgres
    allows multiple NULLs, which is the desired behavior. Case-insensitive uniqueness would need a
    unique index on `(account_id, lower(name))`.
- **Write paths that list columns explicitly** (`PostgresWalletRepository.save`,
  `batch-writer.insertWallets`, `002_seed_treasury_wallets.sql`) keep working against a
  nullable or defaulted column without changes, the same as with `version` in migration 004.
- **Order of steps:** the migration must be applied **before** the backend version that reads
  or writes `name` is deployed. The column is additive, so the old code still runs against the
  new schema; deploying the migration first is safe.

---

## 7. Risks and Dependencies

| # | Risk | Impact | Mitigation |
|---|---|---|---|
| R1 | `Wallet.credit()`/`debit()` not updated to carry `name` | Name silently lost after any remittance (in-memory/tests) | Update both methods; add an entity test |
| R2 | No read/list endpoint exists | Customers can't see names after creation, so the business goal is only partly met | Decide scope (Q5) before building |
| R3 | No ownership check on `POST /wallets` (any authenticated user can open a wallet on any `accountId`) | A user could name another customer's wallet. Would also apply to any new rename/list endpoint | Decide the authorization rule (Q6); needed before any rename/list endpoint |
| R4 | `IdempotentDecorator` replays the cached response by key **without comparing request bodies** | Retrying with the same key but a different name returns the original wallet/name without an error | Existing behavior; document it, or add request-hash checking (out of scope) |
| R5 | zod `z.object` strips unknown keys | If the backend and frontend schemas get out of sync, `name` is silently dropped rather than rejected | Deploy the backend first; test end-to-end |
| R6 | Requiring `name` is a breaking contract change | Old frontend or Bruno requests return 400 | Prefer optional, or coordinate the release |
| R7 | No automated test covers `PostgresWalletRepository` SQL | SQL mistakes (missing `name` in SELECT, wrong `$n`) only show up at runtime | Manual or integration check via Bruno / docker-compose |
| R8 | Migration runner has no version tracking | A non-idempotent migration breaks every later run | Use `IF NOT EXISTS`; keep any backfill idempotent |
| R9 | Path-param route (`/wallets/:id`) vs. `metrics.ts` assumption | Unbounded Prometheus label cardinality | Use the route template instead of `req.path` if rename is added |
| R10 | Name is user-provided free text | Stored XSS (React escapes by default, so low risk in this UI), PII in names, logging of names | Length limit and trim; decide on allowed characters (Q2) |
| R11 | `UNIQUE(account_id, currency)` already makes a wallet unique per currency | A name may add little value under today's rules | Confirm with product (Q7) |

**Dependencies:** Postgres migration → backend deploy → frontend deploy. Swagger and Bruno need
updating in the same backend change. There is no dependency on Redis, RabbitMQ, Kafka, Mongo or
Elasticsearch: wallet names aren't part of any event or read model today. If product wants the
name in the remittance search index, that becomes a further dependency.

---

## 8. Open Questions Requiring Engineering/Product Decisions

The brief says not to assume undocumented business rules, so each of these is left open.

1. **Required or optional?** Must every new wallet have a name, or is it optional?
2. **Validation rules:** min/max length? Allowed characters (emoji, unicode, punctuation)? Trim
   whitespace? Collapse internal whitespace?
3. **Existing wallets:** leave unnamed (NULL plus a UI fallback), or backfill a generated default?
   If backfilling, what format?
4. **Uniqueness:** must names be unique per account? Case-sensitive or not? Across
   active and closed wallets?
5. **Scope of v1:**
   - set the name only at creation;
   - rename later (needs `PATCH`);
   - list wallets with names (needs `GET`)?
   Without the list, the name is visible only once.
6. **Authorization:** who may name a wallet: only the owning account's user, or anyone opening
   it (the current UI lets users open wallets for recipients)? Should this feature introduce the
   missing ownership check on `/wallets`?
7. **Product value under the current constraint:** with one wallet per currency per account,
   is the goal mainly a friendly label? Or is the `UNIQUE(account_id, currency)` rule expected to
   be relaxed later, so that one account can hold several USD wallets? That would be a much
   larger change: `findByAccountIdAndCurrency` drives remittances.
8. **Treasury wallets:** should system treasury wallets get names (e.g. `"Treasury USD"`), or stay
   NULL?
9. **Where else is the name shown?** Remittance form/history, the Elasticsearch remittance index,
   events (`remittance.completed`)?
10. **Idempotency semantics:** is replaying the original response for the same key with a
    different name acceptable (R4)?
11. **Audit:** do renames need history (e.g. a `wallet.renamed` outbox event)?
12. **Domain modelling:** a `WalletName` value object, or DTO-level validation only?

---

## 9. Suggested Implementation Steps

Assumes the minimal scope (name set at creation, optional, nullable column). Adjust once the
open questions are answered.

1. **Settle the decisions:** answer Q1–Q7 at least; freeze the validation rules and the v1 scope.
2. **Database:** add `006_add_wallet_name.sql` with
   `ALTER TABLE wallets ADD COLUMN IF NOT EXISTS name …` (plus any CHECK, UNIQUE index or
   backfill from the Q2–Q4 answers). Apply it locally via the project's `db-migrate` flow.
3. **Domain:** add `name` to `Wallet` (constructor, getter, `credit`/`debit` pass-through);
   optionally add a `WalletName` value object. Update `wallet.test.ts`.
4. **Application:** update `OpenWalletInput` (zod + constructor), `OpenWalletOutput` and
   `OpenWalletUseCase`. Update/add tests in `open-wallet-input.test.ts` and
   `open-wallet-use-case.test.ts`.
5. **Infrastructure:** update `PostgresWalletRepository` (row type, mapper, INSERT/UPSERT,
   SELECTs) and `seed-treasury-wallets.ts`. Optionally update the seed tooling.
6. **Interface/docs:** update `swagger.ts` and the Bruno `Open Wallet` requests; review
   `AGENTS.md` / `docs/` wallet descriptions.
7. **Verify the backend:** `npm test`, lint, then an end-to-end check against Postgres
   (docker-compose plus Bruno): create a wallet with and without a name, send a remittance, and
   confirm the name survives it.
8. **Frontend:** update `lib/schemas/wallet.ts`, then add the Name field and the success-card row
   in `open-wallet-form.tsx`. Read the bundled Next.js docs first (per `AGENTS.md`). Verify with
   `npm run lint`, `npm run build` and a manual run against the updated backend.
9. **Release:** migration → backend → frontend.
10. **Follow-up (if in scope):** `GET /wallets` (with an ownership check) and a frontend wallet
    list; then `PATCH /wallets/:id` (with an ownership check, idempotency, and a fix for the
    `metrics.ts` route label) and a rename UI.

---

## Completion Checklist

- [x] Current wallet creation flow understood (backend and frontend traced end to end)
- [x] All affected layers identified (domain, application, infra, interface, DB, frontend)
- [x] Required changes documented, with file and function references
- [x] Technical risks and unknowns identified (Risks R1–R11, Questions 1–12)
- [x] No source code or database changes made
