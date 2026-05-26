# Tests — CLAUDE.md

## Test Stack

- **Unit/Integration**: Vitest (not Jest)
- **E2E**: Playwright
- **Mocks**: Vitest built-in mocking + hand-written mocks in `tests/mocks/`
- **Fixtures**: `tests/fixtures/` — factory functions, not static JSON

## Structure

```
tests/
  mocks/
    drivers/             # Mock implementations of src/backend/drivers/*
    db.ts                # In-memory DB mock for unit tests
    cache.ts             # In-memory cache mock for unit tests
  fixtures/
    users.ts             # createTestUser(), createTestUsers(n)
    billing.ts           # createTestInvoice(), etc.
  integration/
    api/                 # Tests that hit real Express app + test DB
    db/                  # Tests for complex queries and transactions
    cache/               # Tests for cache behavior with real Redis
  e2e/
    auth.spec.ts         # Login, signup, logout flows
    dashboard.spec.ts    # Main app flows
    ...
  helpers/
    setup.ts             # Global test setup (DB provisioning, env)
    api-harness.ts       # Supertest wrapper for integration tests
```

## Unit tests (colocated)

Unit tests live next to source as `*.test.ts`. They test a single function or module in isolation.

```ts
// src/backend/services/billing.test.ts
import { describe, it, expect, vi } from 'vitest';
import { billingService } from './billing';

describe('billingService.calculateTotal', () => {
  it('applies discount to line items', () => {
    // ...
  });
});
```

- Mock external dependencies (DB, cache, drivers) — test logic only.
- Use fixtures from `tests/fixtures/` for test data, not inline objects.
- No network calls, no filesystem, no DB in unit tests.

## Integration tests

Test real interactions between components. Use the test database and test Redis instance.

- `pnpm test:setup` must run before first integration test (provisions Docker containers).
- Each test file resets its DB state in `beforeEach` using transaction rollback.
- Use `api-harness.ts` for HTTP-level tests against the Express app.
- Test realistic scenarios: create data, perform action, verify side effects.

## E2E tests

Playwright tests against the full running app.

- Authenticate via API shortcut (not clicking through login every test) — see `helpers/auth.ts`.
- Use data-testid attributes for selectors, not CSS classes.
- Test critical paths only — not every edge case. Edge cases belong in unit/integration.
- Screenshots on failure are saved to `tests/e2e/screenshots/`.

## Writing tests for new features

1. **Unit test the service logic** — mock DB and external calls
2. **Integration test the API endpoint** — real DB, verify response + side effects
3. **E2E test the user flow** (if user-facing) — happy path only
4. Run `pnpm test:unit` to verify, then `pnpm test:integration` for DB-touching code

## Common mistakes to avoid

- Don't use `test.only` or `describe.only` in committed code.
- Don't assert on auto-generated values (IDs, timestamps) — assert on shapes or relationships.
- Don't mock what you're testing. If you're testing the DB layer, use the real test DB.
- Don't share mutable state between tests. Each test gets its own data.
