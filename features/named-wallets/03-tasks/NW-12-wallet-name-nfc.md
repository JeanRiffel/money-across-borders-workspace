# NW-12: `WalletName` normalizes to NFC

| | |
|---|---|
| **Spec** | [SPEC-F1](../02-specs/specifications.md#spec-f1-wallet-name-on-creation-backend--database) |
| **Repo** | `money-across-borders` |
| **Depends on** | NW-02 |
| **Decisions** | Q2, Q4, Q13 |
| **Status** | Not started |

## Goal

Make names that look identical compare and store identically (Q13). `"É"` can arrive as one code
point (`U+00C9`) or as `"E"` + combining acute accent (`U+0301`); today `WalletName` keeps whichever
it receives, so Q4's uniqueness would treat them as two names. This task adds Unicode NFC
normalization to the value object from NW-02, before NW-03 starts storing names.

## Acceptance criteria

1. `WalletName.from()` normalizes the input to **NFC first**, before the whitespace handling and
   validation from NW-02. `getValue()` returns the NFC form: `"é"` → `"é"`.
2. Two names that differ only in composed vs decomposed form are equal (`equals`), including
   with different case: `"Épargne"` equals `"éPARGNE"`.
3. Every NW-02 rule still holds, and the existing NW-02 tests keep passing unchanged. Grapheme
   counting is unaffected (a decomposed `"é"` was already 1 grapheme); 40 decomposed `"é"` are still
   accepted and 41 rejected.
4. NFC only: no compatibility folding (NFKC). `"ﬁ"` (`U+FB01`) and full-width digits are kept as
   typed.

## References

- Value object: `src/domain/wallet/value-objects/wallet-name-value-object.ts` (`from()`).
- Existing tests: `__tests__/domain/value-objects/wallet-name-value-object.test.ts`. Add the new
  cases there; don't change the NW-02 ones (their hashes are in the NW-02 log, and a changed hash
  must be explained in RED).
- Why the database index stays the authority on case: Q14.

## Out of scope

- The frontend copy of the rule (NW-10 criterion 2).
- Backfilled names (`"<CURRENCY> Wallet"`) are ASCII, so migration 006 needs no NFC step.

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
