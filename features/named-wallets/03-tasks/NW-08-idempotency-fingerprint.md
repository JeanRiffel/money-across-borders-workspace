# NW-08: Reject a reused Idempotency-Key with a different body

| | |
|---|---|
| **Spec** | [SPEC-T3](../02-specs/specifications.md#spec-t3-idempotency-request-fingerprint-validation) |
| **Repo** | `money-across-borders` |
| **Depends on** | – |
| **Decisions** | Q10 |
| **Status** | Not started |

## Goal

Close discovery risk R4: today, retrying with the same Idempotency-Key and a different body (e.g.
another wallet name) silently returns the original response. Per Q10 that must be rejected.

## Acceptance criteria

1. Same key, same body: the original response is replayed, as today.
2. Same key, different body: rejected with **422** and a clear message, and the use case doesn't
   run. This applies to every endpoint behind `IdempotentDecorator` (accounts, wallets,
   remittances), each controller mapping the new error to 422.
3. The fingerprint ignores differences that don't change meaning: key order in the body, and the
   `idempotencyKey` field itself.
4. It works with every idempotency repository: in-memory, Redis (production) and Postgres (its
   `idempotency_records.request_hash` column already exists in `001_init_schema.sql`).
5. A record stored before this change, which has no fingerprint, is replayed as today, so a deploy
   doesn't turn in-flight retries into errors. *(Proposal made while writing the task.)*
6. Existing idempotency tests still pass, plus new mismatch tests. ADR 0003 describes the
   fingerprint and its normalization.

## References

- `src/application/shared/idempotency/idempotent-decorator.ts:11`.
- `src/infra/persistence/redis/redis-idempotency-repository.ts`.
- Tests: `__tests__/application/shared/idempotency/idempotent-decorator.test.ts`,
  `__tests__/infrastructure/idempotency-repository.test.ts`,
  `__tests__/infrastructure/redis-idempotency-repository.test.ts`,
  `send-remittance-idempotency.test.ts`.

## Out of scope

- Changing the in-flight/claim semantics.

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
