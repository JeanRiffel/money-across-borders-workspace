# Named Wallets

Customers can assign a name to their wallets, so they can identify and distinguish them.

| | |
|---|---|
| **Status** | Specifications proposed. Awaiting engineering approval. |
| **Repositories** | `money-across-borders` (backend), `money-across-borders-web` (frontend) |
| **Last updated** | 2026-10-13 |

## Stages

| Stage | Documents | Status |
|---|---|---|
| 1. Discovery | [brief](01-discovery/brief.md) · [report](01-discovery/report.md) | Done |
| 2. Specifications | [brief](02-specs/brief.md) · [specifications](02-specs/specifications.md) | Proposed, awaiting approval |
| 3. Tasks | [03-tasks/](03-tasks/) | Not started (waits for spec approval) |

## Specifications at a glance

| ID | Specification | Type | Complexity | Decision |
|---|---|---|---|---|
| F1 | Wallet name on creation (backend + database) | Feature | Medium | Pending |
| F2 | Wallet name in the Open Wallet form | Feature | Low | Pending |
| F3 | List wallets with names *(conditional on Q5)* | Feature | Medium | Pending |
| F4 | Rename a wallet *(conditional on Q5)* | Feature | Medium | Pending |
| T1 | Wallet ownership authorization | Technical | Medium | Pending |
| T2 | Metrics route-label normalization | Technical | Low | Pending |
| T3 | Idempotency request fingerprint validation | Technical | Medium | Pending |
| T4 | Postgres wallet repository integration tests | Technical | Medium | Pending |
| T5 | Frontend test harness | Technical | Medium | Pending |

## Open questions

All 12 questions (Q1–Q12) are unanswered. See
[specifications.md → Open Questions](02-specs/specifications.md#open-questions).
Q5 (scope of v1) and Q7 (what a name adds, given one wallet per currency per account) should be
answered first.

| Question | Owner | Answer |
|---|---|---|
| Q1–Q12 | *unassigned* | *open* |
