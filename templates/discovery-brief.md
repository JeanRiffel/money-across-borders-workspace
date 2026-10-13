# Technical Discovery: <Feature Name>

## Task

<One or two sentences describing what customers or the system must be able to do.>

## Business Objective

<Why this matters to the business or the customer.>

## Discovery Objective

Investigate the changes required across the backend,
frontend, and database to support <feature>.

Do not implement the feature.

## Repositories

- money-across-borders (backend)
- money-across-borders-web (frontend)

## Investigation Scope

### Backend

- Identify the relevant domain entities.
- Identify the affected flows (use cases).
- Identify the API endpoints involved.
- Check whether the current schema already supports the change.
- Identify required changes to entities, DTOs, repositories,
  validation, and persistence.
- Identify existing tests that may require updates.

### Frontend

- Identify the affected flows.
- Identify the components responsible for those flows.
- Identify where new data should be collected.
- Identify where new data should be displayed.
- Identify API contracts that may require changes.

### Database

- Inspect the current schema of the affected tables.
- Determine whether a database migration is required.
- Evaluate compatibility with existing records.

## Constraints

- Do not modify any source code.
- Do not modify the database.
- Do not create migrations.
- Do not implement the feature.
- Do not assume business rules that are not documented.
- Clearly identify uncertainties and missing information.

## Expected Output

Produce a technical discovery report containing:

1. Current behavior
2. Proposed behavior
3. Affected components and files
4. Required backend changes
5. Required frontend changes
6. Database implications
7. Risks and dependencies
8. Open questions requiring engineering decisions
9. Suggested implementation steps

Reference existing files and relevant functions
when describing the required changes.

Save the report as `features/<feature-slug>/01-discovery/report.md`.

## Completion Criteria

The discovery is complete when:

- The current behavior of the affected flows is understood.
- All affected layers have been identified.
- Required changes are documented.
- Technical risks and unknowns are identified.
- No source code or database changes have been made.
