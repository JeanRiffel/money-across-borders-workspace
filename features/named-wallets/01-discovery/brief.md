# Technical Discovery: Named Wallets

## Task

Customers must be able to assign a name to their wallets.

## Business Objective

Allow customers to identify and distinguish their wallets
by assigning meaningful names.

## Discovery Objective

Investigate the changes required across the backend,
frontend, and database to support named wallets.

Do not implement the feature.

## Repositories

- money-across-borders (backend)
- money-across-borders-web (frontend)

## Investigation Scope

### Backend

- Identify the wallet domain entity.
- Identify the wallet creation flow.
- Identify the API endpoints involved.
- Check whether the Wallet table already contains a name field.
- Identify required changes to entities, DTOs, repositories,
  validation, and persistence.
- Identify existing tests that may require updates.

### Frontend

- Identify the wallet creation flow.
- Identify the components responsible for wallet creation.
- Identify where the wallet name should be collected.
- Identify where the wallet name should be displayed.
- Identify API contracts that may require changes.

### Database

- Inspect the current Wallet table schema.
- Determine whether a database migration is required.
- Evaluate compatibility with existing wallet records.

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

## Completion Criteria

The discovery is complete when:

- The current wallet creation flow is understood.
- All affected layers have been identified.
- Required changes are documented.
- Technical risks and unknowns are identified.
- No source code or database changes have been made.
