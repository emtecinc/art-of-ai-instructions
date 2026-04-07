# SalesforceSessionRefreshMiddleware Utility

## When to Use
- In tests that may run longer than session timeout due to idle time
- Prevent "Session expired" errors during long test runs

---

## When NOT to Use
- ❌ During impersonation → never register while logged in as another user
- ❌ As replacement for global-setup.ts → still need initial authentication
- ❌ Multiple times on same page → register once per test page
- ❌ Short test runs → unnecessary for tests with idle time < 15 minutes

---

## Import
```typescript
import { SalesforceSessionRefreshMiddleware } from 'playwright-custom-core';
```

---

## Methods

### `register(page: Page): void`
Attach session expiry detection listener to page. Call once per test.

```typescript
const sessionRefresh = new SalesforceSessionRefreshMiddleware();
sessionRefresh.register(page);
```

---

## How It Works

1. **Attach listener** → monitors `framenavigated` events
2. **Track last valid URL** → stores last known Salesforce page URL
3. **Detect expiry** → if redirected to `/visualforce/session` → session expired
4. **Acquire lock** → file-based mutex on `.auth/salesforce.json` (cross-worker safe)
5. **Check freshness** → if another worker already refreshed → reload cookies from file
6. **Refresh if needed** → reset connection → JWT exchange → navigate to frontdoor.jsp → save new session
7. **Navigate back** → return to last valid URL
8. **Release lock** → allow other workers to proceed

---

## Usage Pattern

### Standard Pattern
```typescript
import { test } from '@playwright/test';
import { SalesforceSessionRefreshMiddleware } from 'playwright-custom-core';

test('long-running test', async ({ page }) => {
  const sessionRefresh = new SalesforceSessionRefreshMiddleware();
  sessionRefresh.register(page); // Register once at start of test
  
  // ... test actions that may trigger session expiry ...
  
  // Middleware automatically handles any session expiry
});
```

### With beforeEach
```typescript
import { test } from '@playwright/test';
import { SalesforceSessionRefreshMiddleware } from 'playwright-custom-core';

let sessionRefresh: SalesforceSessionRefreshMiddleware;

test.beforeEach(async ({ page }) => {
  sessionRefresh = new SalesforceSessionRefreshMiddleware();
  sessionRefresh.register(page);
});

test('test 1', async ({ page }) => {
  // Auto-recovers from session expiry
});

test('test 2', async ({ page }) => {
  // Auto-recovers from session expiry
});
```

---

## Impersonation Safety

**CRITICAL**: Never register while impersonated. The middleware refreshes the **admin** JWT, which would corrupt the impersonated session.

---

## Rules

- ✅ **Register once per test** — in test body or `beforeEach`
- ✅ **Admin sessions only** — never register during impersonation
- ✅ **Call early** — register before any Salesforce navigation
- ✅ **Trust automatic recovery** — middleware handles everything silently
- ❌ **Don't register multiple times** — one registration per page
- ❌ **Don't use as setup replacement** — still need global-setup.ts
- ❌ **Don't register on impersonated pages** — corrupts impersonated session

---

## Complete Example

**Test with session refresh**:
```typescript
import { test, expect } from '@playwright/test';
import { SalesforceSessionRefreshMiddleware } from 'playwright-custom-core';
import { CreateAccountWorkflow } from '../workflows/create-account-workflow';

test.describe('Long-running account tests', () => {
  let sessionRefresh: SalesforceSessionRefreshMiddleware;
  let workflow: CreateAccountWorkflow;

  test.beforeEach(async ({ page }) => {
    sessionRefresh = new SalesforceSessionRefreshMiddleware();
    sessionRefresh.register(page);
    
    workflow = new CreateAccountWorkflow(page);
  });

  test('should handle multiple account operations', async () => {
    // Test may run for hours — session auto-refreshes if needed
    for (let i = 0; i < 100; i++) {
      await workflow.createAccount(`Account_${i}`);
      await workflow.verifyAccountCreated(`Account_${i}`);
    }
  });

  test('wait for a process to execute', async () => {
    // Test may have idle time out
    await new Promise(resolve => setTimeout(resolve, 900_000));
  });
});
```