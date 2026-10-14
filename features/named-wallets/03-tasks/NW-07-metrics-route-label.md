# NW-07: Metrics route-label normalization

| | |
|---|---|
| **Spec** | [SPEC-T2](../02-specs/specifications.md#spec-t2-metrics-route-label-normalization) |
| **Repo** | `money-across-borders` |
| **Depends on** | – |
| **Status** | Not started |

## Goal

Stop using the raw `req.path` as the Prometheus `route` label, so a future route with a path
parameter (e.g. `/wallets/:id` for the deferred rename) can't create one label per id. No v1 route
needs it, so this is preventive and can run at any time.

## Acceptance criteria

1. Requests to a parameterized route produce one `route` label value, the template (e.g.
   `/wallets/:id`), whatever ids are used.
2. Static routes keep their current label values (e.g. `/wallets` stays `/wallets`, including for
   routers mounted with a prefix), so dashboards and alerts keep working.
3. Requests that match no route get one fixed label, **`unmatched`**, never the raw path.
4. A unit test checks the label for a static route, a parameterized route and an unmatched request.
5. The comment that says routes are all static is replaced by one describing the new rule.

## References

- `httpMetricsMiddleware`: `src/infra/observability/metrics.ts:72-86`.

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
