# TDD workflow

How each implementation task is built by several coding agents: **RED → GREEN → REFACTOR → VERIFY**.
Global rules (safety, approvals, Git) live in [`AGENTS.md`](../../AGENTS.md) and take precedence over
this file.

## 1. Scope

- Applies to every task in `features/<feature-slug>/03-tasks/`, for example
  [Named Wallets](../../features/named-wallets/03-tasks/README.md).
- The task file is the contract (spec link, acceptance criteria, files to touch) and holds the
  evidence in its `## TDD log` section.
- **Tooling tasks** with no behavior to test first (e.g. a test harness) say so in their task file and
  skip RED. Example: [NW-09](../../features/named-wallets/03-tasks/NW-09-frontend-test-harness.md).
- **Characterization tasks** describe what existing code already does, so their RED tests are
  expected to *pass*. A failing test there is a bug in the current code: log it, and GREEN fixes it.
  If all pass, GREEN and REFACTOR have nothing to do and the task goes straight to VERIFY. Example:
  [NW-01](../../features/named-wallets/03-tasks/NW-01-postgres-wallet-repository-tests.md).

## 2. Roles

| Role | Phases | Current agent |
|---|---|---|
| Test author | RED | Codex |
| Implementer | GREEN, REFACTOR | Claude |
| Verifier | VERIFY | Codex |

The assignment may change; the invariant may not: **the verifier is never the agent that wrote the
production code.** Whichever agent holds a role follows this file as written.

## 3. Before RED

A task may enter RED only when:

- its feature's specs are approved (see the feature's `README.md`);
- none of its acceptance criteria depends on an open question. A test may only encode a rule that the
  spec states or that an answered open question decides;
- the target repository has a test runner. The frontend has none until SPEC-T5
  ([NW-09](../../features/named-wallets/03-tasks/NW-09-frontend-test-harness.md)) lands, so frontend
  tasks wait for it.

Commands come from each repository's own `AGENTS.md`
([backend](../../../money-across-borders/AGENTS.md), [frontend](../../../money-across-borders-web/AGENTS.md)).
Backend: `npm test`; `npm run test:integration` needs Postgres. "Full suite, lint and format check"
below means the repository's commands for those three.

## 4. Phases

Phases run one at a time, in order. A phase finishes, and is logged, before the next starts. The agent
that finishes a phase sets the task's **Status** (in the task file and in the feature's
`03-tasks/README.md`) to the next phase.

### RED (test author)

**Entry:** the conditions in section 3 hold.

**Steps:**

1. Read the approved spec, the task's acceptance criteria and the relevant existing tests.
2. Create the task branch in the code repo from an up-to-date `main`
   (e.g. `feat/nw-02-wallet-name-value-object`). One task at a time: the hand-off is the working tree.
3. Write tests that encode the acceptance criteria. Update existing tests whose setup no longer matches
   the new contract.
4. Run them. Confirm they fail **for the expected reason**: missing behavior, not a typo or broken
   import. For a brand-new module, "Cannot find module" **for that module only** is a valid failure.
5. Record the SHA-256 of each RED test file. A later RED round records new hashes, which replace the
   earlier ones for every later check.

**Must not:** write or change production code; invent rules the spec doesn't state; proceed when an
acceptance criterion is ambiguous (stop, log it, ask a human; see section 6).

**Exit:** tests for every acceptance criterion exist and fail for the expected reason (or pass, for
characterization tasks); the log maps criteria to tests and holds the failing output and hashes.

### GREEN (implementer)

**Entry:** RED is logged. The test hashes match the latest RED entry.

**Steps:**

1. Read the spec and the RED tests.
2. Write the **minimum** production code that makes them pass.
3. Run the full suite, lint and format check.
4. Check the test hashes again.

**Must not:** edit, delete or skip RED tests; expand scope; add abstractions the tests don't need. If a
test looks wrong, stop and log why; the task goes back to RED.

**Exit:** all tests pass, lint and format check pass, hashes unchanged, files changed are logged.

### REFACTOR (implementer, optional)

**Entry:** GREEN is logged and the suite passes.

**Steps:**

1. Review the code GREEN wrote or changed for readability and structure against the repository's
   conventions.
2. Improve it where that is justified, **without changing observable behavior**.
3. Run the full suite, lint and format check, and check the test hashes.
4. Log either the changes with the reason for each, or `No refactoring needed: <reason>`.

**Must not:** change tests; add behavior; touch production code that GREEN didn't touch (tidying
neighboring code is a separate task); add abstractions without a stated reason.

The implementer may do REFACTOR right after GREEN, in the same session, but logs it as its own entry.

**Exit:** suite, lint and format check pass; hashes unchanged; the decision is logged.

### VERIFY (verifier)

**Entry:** REFACTOR is logged (or GREEN, for a characterization task that skipped both).

**Steps:**

1. Check the test hashes against the latest RED entry.
2. Re-run the full suite, lint and format check.
3. Check the code against every acceptance criterion and against the code repository's `AGENTS.md`:
   architecture rules and, in the backend, the invariants in
   [`docs/invariants.md`](../../../money-across-borders/docs/invariants.md).
4. Log *pass*, or numbered findings (`V1`, `V2`, …), each with the file and line, the affected
   criterion, and the phase it goes back to (section 6).

**Must not:** change production code or tests, or fix anything silently.

**Exit, on *pass*:** set the task to *Done*, commit the code on the task branch and open a PR in the
code repo; then commit the task log in this workspace on its own branch and open a PR. This is the
only step that commits without a separate request (see [`AGENTS.md` → Git](../../AGENTS.md#git)).

## 5. Hand-off

The hand-off is the task file's `## TDD log` plus the uncommitted working tree on the task branch.
**Nothing is committed until VERIFY passes.** The backend's pre-commit hook runs `npm test`, so an
earlier commit would be blocked; never bypass it with `--no-verify`.

Each entry should be readable by a human reviewer and by the next agent without other context. Use
this shape, and leave out fields the task header already gives (feature, spec):

```markdown
### <PHASE> [<round>] (<agent>)

- Date: <YYYY-MM-DD>.
- Branch (<repo>): `<branch>`; changes uncommitted.
- Test integrity: SHA-256 of each test file before and after (RED: the new hashes).
- Files created or changed: <paths, one line on what changed>.
- Commands and results: <command>: exit <code>, <counts or the relevant output>.
- Findings, blockers or limitations: <or "none">.
- Next: <PHASE>, by <agent>.
```

Entries written before this template (e.g. NW-02) keep their format.

## 6. Failures and retries

| Situation | Goes to |
|---|---|
| A RED test is wrong (GREEN or VERIFY explains why) | RED |
| GREEN's code fails tests | GREEN continues |
| REFACTOR causes a regression | REFACTOR fixes it or reverts to the GREEN state |
| VERIFY: missing test coverage for a stated rule | RED, then GREEN |
| VERIFY: wrong or incomplete behavior | GREEN |
| VERIFY: a maintainability problem in the new code, behavior correct | REFACTOR |
| An ambiguous or conflicting requirement, in any phase | Stop. Record it as an open question in the feature's spec and ask a human |

- **Every repeat gets a new entry**, never an edit to an earlier one: `RED 2`, `GREEN 2`,
  `REFACTOR 2`, `VERIFY 2`, and so on. A correction to an earlier entry goes in the newer one.
- **Retry limit:** a task that would need a fourth round of any phase (`GREEN 4`, …) stops; ask a human
  before starting it.

## 7. Human approval

Humans decide, and agents don't:

- approval of a feature's specs, before any task exists;
- answers to open questions, and anything escalated under section 6;
- merging the PRs opened when VERIFY passes.

Other than the VERIFY-pass step, nothing is committed or pushed without a request.

## 8. Status values

*Not started* → *RED* → *GREEN* → *REFACTOR* → *VERIFY* → *Done*, and back to an earlier phase per
section 6.

## 9. Environment notes

- Codex's sandbox may block local HTTP listeners, which some backend suites start. Run the suite
  outside that restriction, or log the limitation.
- If a full-suite run fails for reasons unrelated to the task (timeouts in a parallel run, for
  example), log it and re-run, e.g. `npm test -- --runInBand`, rather than ignoring it.
