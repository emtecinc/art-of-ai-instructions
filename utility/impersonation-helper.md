# ImpersonationHelper Utility

## When to Use
- Test features from different user perspectives (profiles, permission sets, roles)
- Verify data visibility and record access per user
- Test role hierarchy and sharing rules
- Validate field-level security and object permissions

---

## When NOT to Use
- ❌ For authentication setup → use `SalesforceLoginService` in global-setup
- ❌ During session refresh → never register `SalesforceSessionRefreshMiddleware` while impersonated
- ❌ For external users/communities → only for internal Salesforce users

---

## Import
```typescript
import { ImpersonationHelper } from 'playwright-custom-core';
```

---

## Methods

### `loginAs(page: Page, displayName: string, baseUrl?: string): Promise<void>`
Provides functionality to log-in as 'displayName' user

**Parameters**:
- `page`: Authenticated Playwright page (admin session)
- `displayName`: User's display name (`"First Last"` or `"Last, First"`)
- `baseUrl`: Salesforce org URL (optional, falls back to `BASE_URL` env var)

```typescript
await ImpersonationHelper.loginAs(page, 'Jane Smith');
// Now acting as Jane Smith
// Checks for user 'Jane Smith' as well as 'Smith, Jane'
```

### `logBack(page: Page, baseUrl?: string): Promise<void>`
Log out from impersonated user and return to admin session.

**Parameters**:
- `page`: Impersonated Playwright page
- `baseUrl`: Salesforce org URL (optional, falls back to `BASE_URL` env var)

```typescript
await ImpersonationHelper.logBack(page);
// Back to admin session
```
---

## Usage Patterns

### Standard Pattern
```typescript
import { test, expect } from '@playwright/test';
import { ImpersonationHelper } from 'playwright-custom-core';

test('verify data as standard user', async ({ page }) => {
  // Start as admin
  await page.goto('/lightning/page/home');
  
  // Switch to standard user
  await ImpersonationHelper.loginAs(page, 'Jane Smith');
  
  // Perform actions as Jane Smith
  await page.goto('/lightning/o/Account/list');
  await expect(page.getByText('Private Account')).not.toBeVisible(); // Verify access restrictions
  
  // Switch back to admin
  await ImpersonationHelper.logBack(page);
  
  // Verify as admin
  await page.goto('/lightning/o/Account/list');
  await expect(page.getByText('Private Account')).toBeVisible();
});
```
---

## Parallel Safety

Each Playwright worker has isolated `BrowserContext`:
- Impersonation in Worker A does **not** affect Worker B
- Shared `.auth/salesforce.json` is **intentionally not overwritten** by `logBack()`
- Prevents race conditions in parallel execution

---

## Rules

- ✅ **Static methods only** — `ImpersonationHelper.loginAs()`, never `new ImpersonationHelper()`
- ✅ **Trust auto-recovery** — `logBack()` handles session corruption
- ❌ **Never register session refresh during impersonation** — corrupts impersonated session
- ❌ **Don't log back every time** — call `logBack()` only when required
- ❌ **Don't create new contexts** — operates on same `BrowserContext`
- ❌ **Don't use for Communities users** — only internal Salesforce users
