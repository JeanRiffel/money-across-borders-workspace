# Named Wallets

Customers can assign a name to their wallets, so they can identify and distinguish them.

| | |
|---|---|
| **Status** | Implementation in progress; NW-01, NW-02 and NW-12 Done. |
| **Repositories** | `money-across-borders` (backend), `money-across-borders-web` (frontend) |
| **Last updated** | 2026-10-15 |

## Stages

| Stage | Documents | Status |
|---|---|---|
| 1. Discovery | [brief](01-discovery/brief.md) · [report](01-discovery/report.md) | Done |
| 2. Specifications | [brief](02-specs/brief.md) · [specifications](02-specs/specifications.md) | Approved (2026-10-14) |
| 3. Tasks | [tasks](03-tasks/README.md) | 12 tasks (NW-01–NW-12; NW-12 added 2026-10-15 for Q13); NW-01 Done (VERIFY passed 2026-10-15), NW-02 Done |

## Specifications at a glance

| ID | Specification | Type | Complexity | Decision |
|---|---|---|---|---|
| F1 | Wallet name on creation (backend + database) | Feature | Medium | Approved |
| F2 | Wallet name in the Open Wallet form | Feature | Low | Approved |
| F3 | List wallets with names *(conditional on Q5)* | Feature | Medium | Approved (in v1 per Q5) |
| F4 | Rename a wallet *(conditional on Q5)* | Feature | Medium | Deferred (out of v1 per Q5) |
| T1 | Wallet ownership authorization | Technical | Medium | Approved |
| T2 | Metrics route-label normalization | Technical | Low | Approved |
| T3 | Idempotency request fingerprint validation | Technical | Medium | Approved |
| T4 | Postgres wallet repository integration tests | Technical | Medium | Approved |
| T5 | Frontend test harness | Technical | Medium | Approved |

## Implementation

Tasks are implemented TDD-style: Codex writes the failing tests (RED), Claude makes them pass
(GREEN) and optionally refactors (REFACTOR), Codex verifies. See the
[TDD workflow](../../docs/workflows/tdd.md).

## Open questions

Full text in [specifications.md → Open Questions](02-specs/specifications.md#open-questions).

| Question | Owner | Answer (2026-10-14 unless stated) |
|---|---|---|
| Q1 | Jean Riffel | Required |
| Q2 | Jean Riffel | 1–40 graphemes, trim + collapse whitespace, any Unicode except control characters |
| Q3 | Jean Riffel | Backfill `"<CURRENCY> Wallet"`; column `NOT NULL` |
| Q4 | Jean Riffel | Unique per account, case-insensitive, active and closed wallets |
| Q5 | Jean Riffel | Creation + listing: F1–F3 in v1, F4 deferred |
| Q6 | Jean Riffel | Only the account owner; others get 404; recipient flow removed |
| Q7 | Jean Riffel | Only a friendly label; one wallet per currency stays |
| Q8 | Jean Riffel | Same backfill as other wallets |
| Q9 | Jean Riffel | Only creation confirmation and wallet list |
| Q10 | Jean Riffel | Reject with 422 (do T3) |
| Q12 | Jean Riffel | `WalletName` value object |
| Q13 | Jean Riffel | NFC normalization (2026-10-15; NW-12) |
| Q14 | Jean Riffel | The database index decides case-insensitive uniqueness (2026-10-15) |
| Q11 | *unassigned* | *open* (only blocks F4, deferred) |
