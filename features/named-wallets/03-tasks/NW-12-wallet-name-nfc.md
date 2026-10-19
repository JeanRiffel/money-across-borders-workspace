# NW-12: `WalletName` normalizes to NFC

| | |
|---|---|
| **Spec** | [SPEC-F1](../02-specs/specifications.md#spec-f1-wallet-name-on-creation-backend--database) |
| **Repo** | `money-across-borders` |
| **Depends on** | NW-02 |
| **Decisions** | Q2, Q4, Q13 |
| **Status** | Done |

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
3. Every NW-02 rule still holds. The NW-02 tests keep passing unchanged, except the expected value
   of the two `decomposed accent` cases (see the 2026-10-18 amendment below). Grapheme counting is
   unaffected (a decomposed `"é"` was already 1 grapheme); 40 decomposed `"é"` are still accepted
   and 41 rejected.
4. NFC only: no compatibility folding (NFKC). `"ﬁ"` (`U+FB01`) and full-width digits are kept as
   typed.

## References

- Value object: `src/domain/wallet/value-objects/wallet-name-value-object.ts` (`from()`).
- Existing tests: `__tests__/domain/value-objects/wallet-name-value-object.test.ts`. Add the new
  cases there. Don't change the NW-02 ones beyond the amendment below; the file's current SHA-256
  (NW-02 RED 2) is `69ac2d6cf7784bb1cf70a9994f5df049b796694ebcee10cd5e09500d0cb30b3f`, and RED
  records the new one.
- Why the database index stays the authority on case: Q14.

## Amendment (2026-10-18)

Criterion 1 conflicted with the original criterion 3. The NW-02 `graphemes` table includes
`['decomposed accent', 'e\u{0301}']`, and two tests that iterate it assert that `getValue()` returns
the input as typed: "should accept one grapheme" and "should accept exactly 40 graphemes"
(test file, lines 52-59). Under NFC, `'e\u{0301}'` becomes `'\u{00E9}'`, so both would fail.

Decision (Jean Riffel, 2026-10-18): RED changes **only the expected value** of those two decomposed
cases to the NFC form (`'\u{00E9}'`, and `'\u{00E9}'.repeat(40)`), keeps the row in the shared table
and the input as is, and logs the change and the new file hash. No other NW-02 test changes.

## Notes for RED

- Expected to **fail** before GREEN: criterion 1, criterion 2, and the two amended decomposed cases.
- Expected to **pass** before GREEN (they guard against regressions or NFKC): the 40/41 decomposed
  limits in criterion 3, and criterion 4 (`"\u{FB01}"` and full-width digits such as `"\u{FF11}"`
  kept as typed). Log them as passing by design, not as a RED failure.
- "NFC first" (criterion 1) is not separately observable through `from()`: NFC leaves `Cc`
  characters alone, and the whitespace it rewrites (`U+2000` → `U+2002`, `U+2001` → `U+2003`) is
  still whitespace that collapses to one space. Don't invent an ordering test; the
  `getValue()` and `equals` assertions cover the behavior.
- Backend commands (from its `AGENTS.md`): `npm test`, `npm run lint`, `npm run format:check`.
  Single file: `npx jest __tests__/domain/value-objects/wallet-name-value-object.test.ts`.

## Out of scope

- The frontend copy of the rule (NW-10 criterion 2).
- Backfilled names (`"<CURRENCY> Wallet"`) are ASCII, so migration 006 needs no NFC step.

## TDD log

### RED (Codex)

- Summary: Added NFC output and canonical-equivalence coverage, plus explicit decomposed-accent
  length and compatibility-preservation guards. Changed only the expected values of the two
  existing NW-02 decomposed-accent cases per the 2026-10-18 amendment; their inputs and shared
  table remain unchanged. RED is complete; production code is unchanged.
- Date: 2026-10-18.
- Branch (backend): `feat/nw-12-wallet-name-nfc`; changes uncommitted. Created from up-to-date
  `main` after `git fetch origin main` and `git merge --ff-only origin/main` (already up to date),
  at `bc03470e927f294fe768df761de5af1789d4bc0b`.
- Branch (workspace): `docs/nw-09-tdd-log`; changes uncommitted alongside existing NW-09 edits.
- Test integrity: `__tests__/domain/value-objects/wallet-name-value-object.test.ts` SHA-256:
  - Before: `69ac2d6cf7784bb1cf70a9994f5df049b796694ebcee10cd5e09500d0cb30b3f`.
  - After: `4df00679782bbf07edbe8691ea062a0abca284b4ef6aa04163bdb18f1b826412`.
  - Changed because RED added seven NW-12 cases and amended the two NW-02 expected values
    from decomposed input to explicit NFC literals (`'\u{00E9}'` and `'\u{00E9}'.repeat(40)`).
    This new hash is the integrity baseline for GREEN and later phases.
- Files created or changed:
  - Backend test file above: seven new cases, two expected-value amendments; no other NW-02 changes.
  - This task and `03-tasks/README.md`: RED evidence and GREEN status.
- Criteria mapped to tests and results:
  - Criterion 1: `should return the NFC form of a decomposed accent` fails: expected U+00E9,
    received U+0065 U+0301.
  - Criterion 2: `should equate composed and decomposed names with same case` and
    `... with different case` fail: expected `true`, received `false`. Both tests assert equality
    in both directions (the first assertion fails before the reverse assertion runs).
  - Criterion 3: existing `should accept one grapheme: decomposed accent` and
    `should accept exactly 40 graphemes: decomposed accent` fail only on the amended NFC output
    expectations. New `should accept 40 decomposed accents as 40 graphemes` and
    `should reject 41 decomposed accents as over 40 graphemes` pass by design; the existing
    41-grapheme decomposed rejection and all other NW-02 cases also pass.
  - Criterion 4: `should preserve fi ligature without compatibility folding` and
    `should preserve full-width digits without compatibility folding` pass by design.
- Commands and results:
  - `npx jest __tests__/domain/value-objects/wallet-name-value-object.test.ts`: exit 1;
    **5 failed, 51 passed, 56 total**, one failed suite. All five failures are missing NFC
    behavior, with no compilation or import failures.
  - Repeated the same `npx jest` command outside sandbox restrictions while investigating delayed
    output: exit 1, identical 5 failed / 51 passed result.
  - `node node_modules/jest/bin/jest.js __tests__/domain/value-objects/wallet-name-value-object.test.ts --runInBand`:
    exit 1, identical 5 failed / 51 passed result.
  - `git diff --check` (backend and workspace): exit 0.
- Findings, blockers or limitations: none in the requirements. NFC ordering is not separately
  observable through `from()`, so no artificial ordering test was added. Full-suite, lint and
  format checks belong to GREEN; this phase intentionally leaves the focused suite red.
- Next: GREEN, by Claude.
- For the next agent: verify the new test hash first, then minimally apply NFC in `WalletName.from()`
  before whitespace handling and validation. Preserve all RED tests, including the compatibility
  guards; run the full suite, lint and format check and recheck the hash. Do not change frontend
  code or start NW-03. Keep unrelated NW-09 workspace edits intact; nothing has been committed.

### GREEN (Claude)

- Summary: `WalletName.from()` now applies `normalize('NFC')` before the existing whitespace
  handling and validation; nothing else in the value object changed. All 56 tests in the
  `WalletName` file pass, including the 5 that RED left failing. The full suite, lint and format
  check pass, and the RED test file is unchanged.
- Date: 2026-10-18.
- Branch (backend): `feat/nw-12-wallet-name-nfc`; changes uncommitted.
- Branch (workspace): `docs/nw-09-tdd-log`; changes uncommitted alongside the NW-09 edits.
- Test integrity: `__tests__/domain/value-objects/wallet-name-value-object.test.ts` SHA-256
  `4df00679782bbf07edbe8691ea062a0abca284b4ef6aa04163bdb18f1b826412` before and after GREEN,
  matching RED.
- Files created or changed:
  - `src/domain/wallet/value-objects/wallet-name-value-object.ts`: `from()` chains
    `.normalize('NFC')` ahead of the whitespace `replace` and `trim`; the comment above `from()`
    now says NFC comes first and that it is NFC, not NFKC. New SHA-256
    `2c02e7c40650bb8226c97814c99da1dba1b6956d1d05f7ef304e914e7391c61a`.
  - This task file and `03-tasks/README.md`: GREEN and REFACTOR entries, status VERIFY.
- Commands and results:
  - `npx jest __tests__/domain/value-objects/wallet-name-value-object.test.ts`: exit 0, 56 passed.
  - `npm test`: exit 1, 262 passed, 1 failed, in
    `__tests__/infra/resilience/resilient-http-client.test.ts` (it uses a local fake FX server with
    timing-based timeouts). That file passes on its own (14/14) both with this change and with it
    stashed, so the failure has nothing to do with NW-12. Re-ran per
    [TDD → Environment notes](../../../docs/workflows/tdd.md#9-environment-notes):
  - `npm test -- --runInBand`: exit 0, 52 suites, 263 passed.
  - `npm run lint`: exit 0, 0 errors, 22 warnings, none in the changed file.
  - `npm run format:check`: exit 0.
- Findings, blockers or limitations: the parallel-run flake in `resilient-http-client.test.ts`
  above is pre-existing and outside this task.
- Next: VERIFY, by Codex (REFACTOR logged below).
- For the next agent: check the test hash, then run `npm test` (use `--runInBand` if
  `resilient-http-client.test.ts` times out in parallel), `npm run lint` and
  `npm run format:check`. Q14 still holds: `equals` is an early check only, and the database index
  decides uniqueness, so NFC here is what makes the stored values match. Nothing is committed.

### REFACTOR (Claude)

- Summary: No refactoring needed: GREEN added one call to an existing expression and updated the
  comment that describes it. Nothing else in the file changed.
- Date: 2026-10-18.
- Test integrity: test SHA-256 still `4df00679…6412`; value object SHA-256 still `2c02e7c4…c61a`.
- Commands and results: same runs as GREEN; nothing changed after them.
- Next: VERIFY, by Codex.

### VERIFY (Codex)

- Summary: **Pass.** Independently verified acceptance criteria 1–4 and the approved amendment.
  The full parallel suite, lint and format check pass. Only the value object and its test file
  differ from backend `origin/main`; VERIFY changed neither file.
- Date: 2026-10-18.
- Branch (backend): `feat/nw-12-wallet-name-nfc`.
- Branch (workspace): `docs/nw-12-tdd-log`, isolated worktree from freshly fetched `origin/main`.
- Test integrity: before and after VERIFY, test SHA-256 is
  `4df00679782bbf07edbe8691ea062a0abca284b4ef6aa04163bdb18f1b826412` (matches RED);
  production SHA-256 is `2c02e7c40650bb8226c97814c99da1dba1b6956d1d05f7ef304e914e7391c61a`
  (matches GREEN/REFACTOR).
- Acceptance review:
  - Criterion 1: `src/domain/wallet/value-objects/wallet-name-value-object.ts:23` applies NFC
    before whitespace replacement, trim and validation. `getValue()` returns that stored value.
  - Criterion 2: `WalletName.equals()` compares normalized values case-insensitively; composed
    versus decomposed same-case and mixed-case tests pass in both directions.
  - Criterion 3 and amendment: all original NW-02 cases pass; only the two authorized expected
    values changed, with original decomposed inputs and table preserved. Explicit 40/41
    decomposed-grapheme boundary checks pass.
  - Criterion 4: explicit NFC retains the fi ligature and full-width digits; both guards pass.
  - Architecture/invariants: normalization stays in the domain value object, adds no dependency,
    and changes no balance, ledger, currency, authorization, idempotency or transaction guarantee.
    Q14's database authority is unaffected.
- Files created or changed by VERIFY: this task log, only the NW-12 status row in
  `03-tasks/README.md`, and the feature README status line. NW-09 edits remain outside this PR.
- Commands and results:
  - `npm test` (outside sandbox to allow local listeners): exit 0, **52 suites / 263 tests passed**.
    The known parallel timing flake did not occur; no serial retry was needed.
  - `npm run lint`: exit 0, 0 errors, 22 existing warnings.
  - `npm run format:check`: exit 0, all matched files use Prettier style.
  - `git diff --check` (backend): exit 0.
  - SHA-256 checks before and after review: exact matches above.
- Findings, blockers or limitations: none. Database integration tests are not applicable to this
  domain-only change.
- Next: human review and merge of the backend and workspace PRs; NW-12 is Done.
- For the next agent: use the separate NW-12 PRs. Do not include or commit the pending NW-09
  task or its status row; they remain uncommitted on `docs/nw-09-tdd-log`.
