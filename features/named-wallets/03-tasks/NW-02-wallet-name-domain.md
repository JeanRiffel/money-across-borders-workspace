# NW-02: `WalletName` value object

| | |
|---|---|
| **Spec** | [SPEC-F1](../02-specs/specifications.md#spec-f1-wallet-name-on-creation-backend--database) |
| **Repo** | `money-across-borders` |
| **Depends on** | – |
| **Decisions** | Q2, Q12 |
| **Status** | Done |

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

- Date: 2026-10-14.
- Branch (backend): `feat/nw-02-wallet-name-value-object`, created from updated `main` /
  `origin/main` at `0a722a4b993981b09da485629af431f2cdb68a3d` after `git fetch origin main`.
- Tests added: [`__tests__/domain/value-objects/wallet-name-value-object.test.ts`](../../../../money-across-borders/__tests__/domain/value-objects/wallet-name-value-object.test.ts).
  Existing tests and production files were not changed. Changes are uncommitted.
- Contract selected for GREEN: export `WalletName` from
  `src/domain/wallet/value-objects/wallet-name-value-object.ts`, with `WalletName.from(string)`,
  `getValue(): string` and `equals(other: WalletName): boolean`. Invalid names throw the existing
  `ValidationError` from `src/domain/shared/errors.ts`. Error messages identify an empty/minimum
  length violation, the 40-grapheme maximum, or a control-character violation; tests use regexes
  rather than prescribe exact wording.
- Acceptance criteria → tests (parameterized cases included):

  | Criterion | Test coverage |
  |---|---|
  | 1 — whitespace normalization | `should trim the ends and collapse internal spaces, tabs and newlines`; `should normalize %s before validating controls` covers NBSP, em space, narrow NBSP, ideographic space, line/paragraph separators, CR, VT and FF. |
  | 2 — normalization before length validation, 1–40 graphemes | `should validate the normalized length rather than the input length`; `should reject an empty name after normalization (%j)`; `should accept one grapheme: %s`, `should accept exactly 40 graphemes: %s`, and `should reject 41 graphemes: %s` each cover ASCII, family emoji with ZWJ, a Brazil flag, precomposed accent and decomposed accent. |
  | 3 — allowed Unicode, rejected remaining Cc, allowed ZWJ | `should allow Unicode and preserve %s` covers accents, CJK, emoji, punctuation and standalone ZWJ; `should reject remaining Cc characters: %s` covers NUL, BEL, backspace, DEL and a C1 control at the start, middle and end. Whitespace-control normalization is covered by criteria 1–2. |
  | 4 — domain error, explanatory message and HTTP 400 | `should identify the failed rule for %s` checks `ValidationError` and a rule-specific message for empty, whitespace-only, overlong and control-containing names; `should map the domain error for %s to HTTP 400` invokes `WalletName.from` through a test use case and the real `OpenWalletController.handle`, asserting execution, status and message. It does not require the NW-03 DTO/entity changes. |
  | 5 — normalized value and case-insensitive equality | `should compare normalized names case-insensitively` checks both comparison directions, self-equality, different names, normalized internal whitespace and accented case pairs; `should expose its normalized value without mutating it during comparison` checks preserved casing in both instances. |
  | 6 — factory/value-object style and own adjacent unit test | The new file uses neighboring `describe` / `it` / `expect` conventions; `should expose its normalized value without mutating it during comparison` checks the factory returns a `WalletName`, its accessor and non-mutating comparison. Private constructor/readonly implementation style remains a GREEN/VERIFY code-review check; runtime freezing is not required by the existing value objects. |

- Commands and results (run in the backend):
  - `npm test -- wallet-name`: exit 1; **1 failed suite, 0 executed tests**. The only diagnostic is
    `TS2307: Cannot find module '../../../src/domain/wallet/value-objects/wallet-name-value-object'
    or its corresponding type declarations.` This points to the intended missing production
    module, not another import. Repeated after the test lint correction with the same result.
  - `npm test`: exit 1; **48 passed suites, 4 failed suites; 188 passed tests, 19 failed tests**.
    Besides the expected missing module, the sandbox blocked local HTTP listeners with
    `listen EPERM: operation not permitted 127.0.0.1` in the existing resilience/FX tests;
    a seed determinism test exceeded its 15-second timeout during the parallel run.
  - `npm test -- --runInBand` (outside the sandbox restriction): exit 1; **51 passed suites,
    1 failed suite; all 207 existing tests passed**. The sole failure is the missing
    `wallet-name-value-object` module (`TS2307`). No existing tests were changed or skipped.
  - `npm run lint`: exit 0; **0 errors, 22 existing warnings** in unchanged `src/` files.
    This script only checks production sources.
  - `./node_modules/.bin/eslint __tests__/domain/value-objects/wallet-name-value-object.test.ts`:
    final exit 0, no errors or warnings. An initial `no-unexpected-multiline` finding in the new
    test was corrected before the final checks.
  - `git diff --check`: exit 0.
- Test file SHA-256 for VERIFY (RED tests must remain unchanged):
  `5ae8e35541af32bcd67b2f882c0a094605d0473de1a74a9fdc7a15bbeccf0a94`.
- Coverage gaps / ambiguous criteria: none. The new suite cannot execute its cases until GREEN
  adds the missing module; that is the requested RED failure.

### GREEN (Claude)

- Date: 2026-10-14.
- Branch (backend): `feat/nw-02-wallet-name-value-object`. Changes are uncommitted; the RED test
  file is untouched (SHA-256 still `5ae8e355…0a94`).
- Files changed:
  - **New** `src/domain/wallet/value-objects/wallet-name-value-object.ts`: `WalletName` with a
    private constructor, `from()`, `getValue()`, `equals()` (compares `toLowerCase()`, matching the
    `lower(name)` index planned in NW-04) and `toString()`, in the style of `WalletId`. `from()`
    replaces each `\s+` run with one space and trims, then rejects in this order: empty →
    `\p{Cc}` → more than 40 graphemes (`Intl.Segmenter`, granularity `grapheme`). The value is
    stored as typed (no NFC normalization), as the decomposed-accent test requires.
  - `tsconfig.json`: `lib` gains `"ES2022.Intl"` for the `Intl.Segmenter` types. `target` is
    unchanged; Node 22 provides `Intl.Segmenter` at runtime.
- Results:
  - `npm test -- wallet-name`: 1 suite, **46/46 passed**.
  - `npm test -- --runInBand`: **52/52 suites, 253/253 tests passed**.
  - `npm run lint`: 0 errors, 22 warnings, all pre-existing in unchanged files.
  - `npm run format:check`: all files pass.
  - `npx tsc --noEmit -p .`: 5 errors, the same 5 as on `main` without these changes (missing
    modules in `src/infra/config/message-broker/rabbitmq-*.ts`); none in NW-02 files.
  - `git diff --check`: clean.
- Pending: none. Not committed (VERIFY first).

### VERIFY (Codex)

- Date: 2026-10-14.
- Result: **findings — return to RED**. All existing tests pass, but the Unicode whitespace
  requirement has an uncovered failing case. RED must encode that case before GREEN corrects
  production behavior. VERIFY did not change production code, configuration or tests.
- Integrity:
  - Backend branch remained `feat/nw-02-wallet-name-value-object`; no checkout or commit.
  - `git status --short` shows exactly `M tsconfig.json`, the untracked RED test and the untracked
    GREEN value object, both before and after verification.
  - `sha256sum __tests__/domain/value-objects/wallet-name-value-object.test.ts` matches RED exactly:
    `5ae8e35541af32bcd67b2f882c0a094605d0473de1a74a9fdc7a15bbeccf0a94`.
- Commands and results (backend unless stated otherwise):
  - `npm test -- wallet-name` outside the HTTP-restricted sandbox: exit 0, **46/46 tests**.
  - `npm test -- --runInBand` outside that restriction: exit 0, **52/52 suites, 253/253 tests**.
  - `npm run lint`: exit 0, **0 errors, 22 pre-existing warnings** in unchanged files.
  - `npm run format:check`: exit 0, all matched source files pass.
  - `npx tsc --noEmit -p .`: exit 2, **5 pre-existing errors**, none in NW-02 files.
    Comparison used a `git archive main` snapshot under `/tmp`, with the same installed
    dependencies, compiled by `node_modules/.bin/tsc --noEmit -p <snapshot>`; the worktree and
    branch were not modified. After removing the snapshot path prefix, all five diagnostics
    match exactly. Baseline commit: `0a722a4b993981b09da485629af431f2cdb68a3d`.
    Contrary to GREEN's description that all five are missing RabbitMQ modules, they are:
    - `src/infra/config/database/DatabaseContext.ts:4` and `:6`: TS2314, missing generic type
      argument for `DatabaseStrategy<T>`.
    - `src/infra/config/message-broker/rabbitmq-consumer.ts:1` and `:2`, and
      `rabbitmq-producer.ts:1`: TS2307, missing modules.
  - `npx tsc --noEmit -p . --lib ES2020`: read-only diagnostic check, exit 2; adds TS2339 at
    `wallet-name-value-object.ts:8` (`Intl.Segmenter` types missing) to the five baseline errors.
    This verifies why `ES2022.Intl` is needed without editing `tsconfig.json`.
  - `node -r ts-node/register` with an inline verification probe: exit 0; Unicode
    `\p{White_Space}` matches `\u{0085}`, while `\s` does not. Calling
    `WalletName.from('\u{0085}My\u{0085}Savings\u{0085}')` throws
    `ValidationError: Wallet name must not contain control characters`, rather than returning
    `My Savings`. The same probe confirms precomposed/decomposed accent equality is false.
  - `git diff --check`: exit 0 in backend and workspace.
- Acceptance criteria review:

  | Criterion | Result |
  |---|---|
  | 1 — normalize whitespace | **Finding V1/V2** below: covered examples pass, but Unicode NEXT LINE is missed. Normalization otherwise precedes validation (`wallet-name-value-object.ts:17`). |
  | 2 — 1–40 graphemes | Accepted: empty check at `:19`, `Intl.Segmenter` with grapheme granularity at `:8`, maximum check at `:25`; tests exercise 1/40/41 with ASCII, family emoji, flags and both accent representations. |
  | 3 — reject remaining Cc, allow other Unicode/ZWJ | The `\p{Cc}` rejection at `:22` and allowed-Unicode tests are correct; V1 means one whitespace control incorrectly remains for this check to reject. |
  | 4 — domain error/message/400 | Accepted: existing domain `ValidationError`, messages at `:20`, `:23`, `:26`; real controller mapping exercised by four RED cases. Wiring name into the real DTO/use case remains NW-03. |
  | 5 — normalized accessor/case-insensitive equality | Accepted: accessor at `:32`, `toLowerCase()` comparison at `:37`; tests cover both directions, self, differing names, whitespace and accented case. Canonical Unicode equivalence is discussed below. |
  | 6 — value-object style/immutability/tests | Accepted: private constructor and readonly string at `:11`, factory, no setters or mutation, adjacent unit tests. Matches `WalletId`/`Currency`; runtime freezing is not required. |

- Findings:
  - **V1 — production behavior**, `src/domain/wallet/value-objects/wallet-name-value-object.ts:17`:
    ECMAScript `\s` excludes `U+0085` (NEXT LINE), which belongs to Unicode `White_Space` and
    category `Cc`. Criterion 1 requires Unicode whitespace to be trimmed/collapsed before
    validation; the current factory instead reaches `:22` and rejects the example above.
    Affected criteria: **1**, and the normalization-before-remaining-controls rule in **3**.
  - **V2 — RED coverage gap**, `__tests__/domain/value-objects/wallet-name-value-object.test.ts:30`:
    the whitespace table omits `\u{0085}`, so 46 passing tests do not fully establish criterion 1.
    Add RED cases for leading/trailing and repeated internal NEXT LINE whitespace, plus a
    whitespace-only case, then return to GREEN. This supersedes RED's earlier statement that
    there were no coverage gaps. Affected criteria: **1/3**. Tests remain unchanged in VERIFY.
- Requested assessment:
  - **Accepted — no NFC in equality**, `wallet-name-value-object.ts:38`: `\u{00E9}` and
    `e\u{0301}` remain distinct. Criterion 2 requires each to count as one grapheme, not to be
    equal; criterion 5 requires case-insensitive equality, not canonical-equivalence folding.
    This is consistent with NW-04's planned `lower(name)` expression containing no explicit
    Unicode normalization. Exact PostgreSQL equality still depends on the chosen collation;
    database parity was not established by this domain-only VERIFY and belongs to NW-04.
  - **Accepted — minimal Intl library change**, `tsconfig.json:5`: adding `ES2022.Intl` supplies
    the `Intl.Segmenter` declarations. The read-only comparison above demonstrates necessity;
    target and module settings remain unchanged. Runtime Node is `v22.22.1` and segmentation
    passes the tests.
  - **Coverage**: V2 is a real missing edge case in a stated rule; private constructor/readonly
    and domain dependency direction were verified by source review rather than runtime tests.
- Architecture/invariants: the new object imports only domain errors and uses the standard Intl
  API. No infrastructure dependencies, money/balance/ledger/persistence mutations, authorization
  changes or weakened guaranteed invariants were introduced.
- Commands not executed / execution limitations: **none of the requested commands**. Existing
  TypeScript baseline errors are recorded above. No database integration was needed for NW-02.
- Follow-up: status is **RED** in this task and the task index because V2 must be addressed by
  the RED agent before Claude corrects V1 in GREEN. No acceptance criteria or other tasks changed.

### RED 2 (Codex)

- Date: 2026-10-14.
- Branch (backend): `feat/nw-02-wallet-name-value-object`; existing uncommitted changes preserved.
  Status remains **RED** pending GREEN.
- Tests added to `__tests__/domain/value-objects/wallet-name-value-object.test.ts` for VERIFY
  findings V1/V2 (criteria 1/3, with the empty-name error from criteria 2/4):
  - `should trim NEXT LINE whitespace at both ends`:
    `'\u{0085}My Savings\u{0085}'` → `'My Savings'`.
  - `should collapse repeated internal NEXT LINE whitespace`:
    `'My\u{0085}\u{0085}Savings'` → `'My Savings'`.
  - `should reject a NEXT LINE-only name as empty rather than as a control character`:
    `'\u{0085}\u{0085}'` → `ValidationError` with an empty/minimum-length message, and no
    control-character message.
- Existing tests were neither edited nor removed. Comparing against the pre-edit copy confirms
  that only the three tests above were inserted at the end of the existing `describe` block.
- Commands and results:
  - `npm test -- wallet-name`: exit 1; **3 failed, 46 passed, 49 total**. Only the three new
    NEXT LINE tests fail. The first two unexpectedly throw
    `ValidationError: Wallet name must not contain control characters` at
    `src/domain/wallet/value-objects/wallet-name-value-object.ts:23`. The whitespace-only test
    expects `/empty|at least.*1|1.*40.*grapheme/i`, but receives the same control-character
    message. These are the expected missing-normalization failures, with no syntax, type or
    import errors.
  - `./node_modules/.bin/eslint __tests__/domain/value-objects/wallet-name-value-object.test.ts`:
    exit 0, no errors or warnings.
  - `git diff --check`: exit 0 in backend and workspace.
- New test file SHA-256 for the next GREEN/VERIFY:
  `69ac2d6cf7784bb1cf70a9994f5df049b796694ebcee10cd5e09500d0cb30b3f`.
  This replaces the original RED hash for subsequent test-integrity checks.
- Production code and `tsconfig.json` were not modified; their SHA-256 hashes before and after
  this RED supplement are identical:
  - `wallet-name-value-object.ts`:
    `c61db414395143199b74fa401e2eb5b6ef0f5acbba71613c43352e7b3ebf13ba`.
  - `tsconfig.json`:
    `157bd8cb134dc0260b48bb440e580aa082ff2aff3dddc68928d1b85b8da95f38`.
- No commit, branch switch, acceptance-criteria changes or other-task edits. V2 is now encoded
  in failing tests; Claude can proceed with GREEN to fix V1 while preserving all 49 tests.

### GREEN 2 (Claude)

- Date: 2026-10-14.
- Branch (backend): `feat/nw-02-wallet-name-value-object`; no branch switch, no commit.
- Test integrity: SHA-256 of the test file is
  `69ac2d6cf7784bb1cf70a9994f5df049b796694ebcee10cd5e09500d0cb30b3f` before and after the change.
  `tsconfig.json` is unchanged (`157bd8cb…5f38`).
- Files changed: `src/domain/wallet/value-objects/wallet-name-value-object.ts` only. The
  normalization regex goes from `/\s+/gu` to `/[\s\p{White_Space}]+/gu`, with a comment explaining
  why both are needed: `\s` misses U+0085 (Cc and White_Space), and `\p{White_Space}` alone would
  stop normalizing U+FEFF, which `\s` already covered. Validation order (empty → `\p{Cc}` → length)
  is unchanged, so controls left after normalization are still rejected. New file SHA-256:
  `b3cce1612c4b75db705b3b8f6fecdfb2b413426a702a5cdb5b910a6aeff1e1a1`.
- Results (run without the HTTP-listener sandbox restriction; the resilience/FX suites that start
  local servers ran and passed):
  - `npm test -- wallet-name`: **49/49 passed**.
  - `npm test -- --runInBand`: **52/52 suites, 256/256 tests passed**.
  - `npm run lint`: 0 errors, 22 pre-existing warnings in unchanged files.
  - `npm run format:check`: all files pass.
  - `git diff --check`: clean.
- Correction to the first GREEN entry: of the 5 pre-existing `tsc` errors, 3 are in
  `rabbitmq-*.ts` and 2 in `src/infra/config/database/DatabaseContext.ts`, as VERIFY recorded.
- Pending: none. Ready for VERIFY 2 (Codex).

### VERIFY 2 (Codex)

- Date: 2026-10-14.
- Result: **pass**. Findings V1 and V2 are resolved; no new findings. Status changed to **Done**
  in this task and the task index. Production code, tests and configuration were not edited in
  VERIFY 2; all uncommitted backend changes remain on the original branch.
- Integrity checked before and after verification:
  - Branch: `feat/nw-02-wallet-name-value-object` (no branch switch or commit).
  - Backend `git status --short` contains exactly `M tsconfig.json`, the untracked test file and
    the untracked value object; no other backend changes.
  - Test SHA-256 matches RED 2:
    `69ac2d6cf7784bb1cf70a9994f5df049b796694ebcee10cd5e09500d0cb30b3f`.
  - `tsconfig.json` SHA-256 matches the previously accepted GREEN configuration:
    `157bd8cb134dc0260b48bb440e580aa082ff2aff3dddc68928d1b85b8da95f38`.
  - Value object SHA-256 matches GREEN 2:
    `b3cce1612c4b75db705b3b8f6fecdfb2b413426a702a5cdb5b910a6aeff1e1a1`.
- Commands and results:
  - `npm test -- wallet-name` outside the HTTP-restricted sandbox: exit 0, **49/49 tests**.
  - `npm test -- --runInBand` outside that restriction: exit 0, **52/52 suites, 256/256 tests**.
    The existing local-HTTP tests executed successfully; no tests were skipped.
  - `npm run lint`: exit 0, **0 errors, 22 pre-existing warnings** in unchanged files.
  - `npm run format:check`: exit 0, all matched production sources pass.
  - `npx tsc --noEmit -p .`: exit 2, exactly the **5 pre-existing diagnostics**, none in NW-02:
    TS2314 at `src/infra/config/database/DatabaseContext.ts:4` and `:6`, and TS2307 at
    `src/infra/config/message-broker/rabbitmq-consumer.ts:1`, `:2` and `rabbitmq-producer.ts:1`.
    The output matches the main-snapshot diagnostics captured during VERIFY exactly after
    stripping the snapshot path prefix. `git rev-parse main` remains
    `0a722a4b993981b09da485629af431f2cdb68a3d`; `git diff main --` for these three source files
    is empty. The TypeScript baseline has not changed.
  - `git diff --check`: exit 0 in both backend and workspace.
- GREEN 2 scope and regression review:
  - A read-only Python check reversed only the regex replacement and its explanatory comment
    in memory. The resulting SHA-256 equals the pre-GREEN-2 production hash from RED 2,
    `c61db414395143199b74fa401e2eb5b6ef0f5acbba71613c43352e7b3ebf13ba`.
    Thus no other production change was introduced in GREEN 2.
  - A read-only `node -r ts-node/register` probe enumerated every code point through `U+10FFFF`.
    The old `\s` set contains 25 code points; `[\s\p{White_Space}]` contains 26, retaining all
    25 and adding only `U+0085`. For every matched character, the real factory normalizes
    leading/trailing and repeated internal whitespace and rejects whitespace-only names as
    empty. An explicit `\u{FEFF}` example returns `My Savings`, preserving prior behavior.
  - The same probe verifies all **59 Cc characters outside the whitespace union** still raise
    a control-character `ValidationError` at the start, middle and end of a name. Probe exit 0.
- Final acceptance-criteria review (references to the final value object):

  | Criterion | Verification |
  |---|---|
  | 1 — whitespace normalization | Pass: union regex and trimming at `wallet-name-value-object.ts:19`, existing whitespace tests, all three NEXT LINE tests and the exhaustive whitespace probe. V1 is resolved. |
  | 2 — 1–40 graphemes after normalization | Pass: empty validation at `:21`, grapheme `Intl.Segmenter` at `:8`, upper limit at `:27`; RED covers 1/40/41 ASCII, family emoji, flags and precomposed/decomposed accents. |
  | 3 — reject remaining Cc, allow Unicode and ZWJ | Pass: Unicode-category rejection at `:24`; allowed-Unicode/ZWJ tests and exhaustive non-whitespace-control probe. NEXT LINE is removed before this check. |
  | 4 — domain error, rule message, HTTP 400 | Pass: `ValidationError` import at `:1`, messages at `:22`, `:25`, `:28`; existing real-controller mapping tests pass, and NEXT LINE-only input gets the empty-name error. |
  | 5 — normalized value, case-insensitive equality | Pass: `getValue()` at `:34`, `equals()` at `:39`, with both-direction/self/different-name/normalized-whitespace/accented-case coverage. The no-NFC assessment from VERIFY remains accepted. |
  | 6 — existing value-object style and adjacent tests | Pass: private constructor/readonly string at `:11`, static factory, no setters or mutation, accessor and adjacent test suite. Matches neighboring `WalletId`/`Currency` conventions. |

- V2 is resolved by the three RED 2 tests, whose hash remains unchanged. No uncovered stated
  behavior was identified in this review. Constructor visibility, readonly storage and domain
  dependency direction are verified through code review as well as behavioral tests.
- Architecture/invariants: only domain errors and standard Intl are used; no infrastructure
  dependency, financial-invariant weakening, authentication change or persistence change.
  The previously accepted `ES2022.Intl` configuration is unchanged.
- Findings: **none**. Requested commands not executed / execution limitations: **none**.
  The five baseline TypeScript errors remain documented above; they are not NW-02 regressions.
