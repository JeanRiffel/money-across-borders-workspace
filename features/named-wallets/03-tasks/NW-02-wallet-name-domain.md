# NW-02: `WalletName` value object

| | |
|---|---|
| **Spec** | [SPEC-F1](../02-specs/specifications.md#spec-f1-wallet-name-on-creation-backend--database) |
| **Repo** | `money-across-borders` |
| **Depends on** | – |
| **Decisions** | Q2, Q12 |
| **Status** | Not started |

## Goal

Add a domain value object that owns the wallet-name rules (Q12), so every write path validates and
normalizes names the same way. This task adds only the value object. The `Wallet` entity starts
using it in NW-03.

## Acceptance criteria

Normalization happens before validation, in this order:

1. Leading and trailing whitespace is removed, and each run of internal whitespace (spaces, tabs,
   newlines and other Unicode whitespace) becomes one space. `"  My \t\n Savings  "` →
   `"My Savings"`.
2. After normalization, a name must be **1 to 40 grapheme clusters** long, counted with
   `Intl.Segmenter` (granularity `grapheme`):
   - `""` and `"   "` are rejected (empty after normalization);
   - a 40-grapheme name is accepted; a 41-grapheme name is rejected;
   - `"👨‍👩‍👧"` counts as 1, a flag such as `"🇧🇷"` counts as 1;
   - `"é"` counts as 1 whether it is precomposed (`U+00E9`) or decomposed (`e` + `U+0301`).
3. Any Unicode is accepted (accents, CJK, emoji, punctuation) **except** control characters
   (category `Cc`, e.g. `U+0000`, `U+0007`, `U+007F`) left after normalization. Format characters
   such as the zero-width joiner (`U+200D`) are allowed, because emoji sequences need them.
4. An invalid name raises a domain error with a message that says which rule failed. It must reach
   the client as **400** through the existing mapping in `OpenWalletController` (it already maps
   `ValidationError` to 400); RED picks the error class and the test asserts it.
5. The value object exposes its normalized value, and equality between two names is
   **case-insensitive** (`"Travel"` equals `"TRAVEL"`), which NW-03 uses for uniqueness (Q4).
6. It follows the style of the existing value objects (factory method, immutable, `getValue()`),
   and has its own unit test next to theirs.

## References

- Existing value objects to follow: `src/domain/wallet/value-objects/wallet-id-value-object.ts`,
  `src/domain/shared/value-objects/currency-value-object.ts`; tests in
  `__tests__/domain/value-objects/`.
- Errors: `src/domain/shared/errors.ts` (`ValidationError` at line 88).
- Where validation belongs: ADR 0009 (shape in zod, business rules in the domain).

## Out of scope

- Changing `Wallet`, DTOs or persistence (NW-03, NW-04).
- The frontend copy of these rules (NW-10).

## TDD log

### RED (Codex)

- Date:
- Tests added/changed:
- Command and failing output (summary):

### GREEN (Claude)

- Date:
- Files changed:
- Results of `npm test`, lint and format check:

### VERIFY (Codex)

- Date:
- Result (pass / findings):
