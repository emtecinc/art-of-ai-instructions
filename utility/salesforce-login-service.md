# SalesforceLoginService Utility (INCOMPLETE)

## Purpose
Establishes authenticated Playwright `BrowserContext` via Salesforce frontdoor.jsp. Used exclusively in `global-setup.ts` to create reusable browser sessions.

---

## When to Use
- In `global-setup.ts` to authenticate once before all tests
- Create browser session with Salesforce cookies
- Generate storage state file for test reuse

---

## When NOT to Use
- ❌ Inside test bodies → tests use saved session automatically
- ❌ For API authentication → use `SalesforceConnection`
- ❌ For session refresh → use `SalesforceSessionRefreshMiddleware`
- ❌ Multiple times per run → authenticate once in global-setup

---

## Import
```typescript
import { SalesforceLoginService } from 'playwright-custom-core';
```

---

## Constructor

```typescript
new SalesforceLoginService(sfConnection?: SalesforceConnection)
```

| Parameter | Default | Notes |
|---|---|---|
| `sfConnection` | `SalesforceConnection.getInstance()` | Override for testing/isolation |

**Standard usage**:
```typescript
const loginService = new SalesforceLoginService(); // Uses singleton
```

---

## Methods

### `login(browser): Promise<BrowserContext>`
Authenticate via JWT + frontdoor.jsp and return authenticated context.

**Parameters**:
- `browser`: Playwright `Browser` instance

**Returns**: Authenticated `BrowserContext` with Salesforce cookies

```typescript
import { chromium } from '@playwright/test';
import { SalesforceLoginService } from 'playwright-custom-core';

const browser = await chromium.launch();
const loginService = new SalesforceLoginService();

const context = await loginService.login(browser);
// Context now has valid Salesforce session cookies
```

---

## Authentication Flow

1. **JWT authentication** → `SalesforceConnection.getConnection()` → get access token
2. **Construct frontdoor URL** → `${instanceUrl}/secur/frontdoor.jsp?sid=${accessToken}`
3. **Navigate to frontdoor** → Salesforce redirects to homepage and sets session cookies
4. **Wait for Lightning shell** → `role="main"` visible → page ready
5. **Return context** → authenticated `BrowserContext` with cookies

---

## Usage Pattern

### In global-setup.ts
```typescript
import { chromium } from '@playwright/test';
import { SalesforceLoginService, SessionManager } from 'playwright-custom-core';

export default async function globalSetup() {
  // 1. Ensure auth directory exists
  SessionManager.ensureDir();
  
  // 2. Launch browser
  const browser = await chromium.launch();
  
  // 3. Authenticate
  const loginService = new SalesforceLoginService();
  const context = await loginService.login(browser);
  
  // 4. Save session to file
  await context.storageState({ path: SessionManager.getStorageStatePath() });
  
  // 5. Cleanup
  await context.close();
  await browser.close();
}
```

### In playwright.config.ts
```typescript
import { defineConfig } from '@playwright/test';
import { SessionManager } from 'playwright-custom-core';

export default defineConfig({
  globalSetup: './global-setup.ts',
  use: {
    storageState: SessionManager.getStorageStatePath(),
    // All tests automatically use the authenticated session
  },
});
```

---

## MFA Handling

If the automation user has MFA enabled, Salesforce will redirect to MFA prompt instead of frontdoor, causing login to fail.

**Solution**: Assign the **"Waive Multi-Factor Authentication for Exempt Users"** permission set to the automation user.

**Error message**:
```
Error: MFA redirect detected at /identity/verify/mfa.
Action required: assign automation user the 'Waive Multi-Factor Authentication for Exempt Users' permission set.
```

---

## Frontdoor.jsp Explained

`frontdoor.jsp` is Salesforce's single sign-on endpoint:
- **URL**: `https://instance.salesforce.com/secur/frontdoor.jsp?sid={accessToken}`
- **Behavior**: Validates access token → sets session cookies → redirects to home
- **Result**: Browser context has valid Salesforce session (no username/password UI interaction)

---

## Error Scenarios

### Invalid credentials
```
Failed to authenticate: SalesforceConnection.getConnection() threw error
```
- Check `SF_USERNAME`, `SF_CLIENT_ID`, `SF_PRIVATE_KEY_PATH`
- Verify user is pre-authorized in Connected App

### MFA redirect
```
MFA redirect detected at /identity/verify/mfa
```
- Assign "Waive Multi-Factor Authentication for Exempt Users" permission set

### Session not persisting
```
Tests fail with "Session expired" or redirect to login
```
- Ensure `playwright.config.ts` references `SessionManager.getStorageStatePath()`
- Check `.auth/salesforce.json` exists after global-setup

---

## Session Reuse

Benefits of `global-setup.ts` authentication:
- **Authenticate once** → all tests reuse session
- **Faster execution** → no login per test
- **Storage state** → `.auth/salesforce.json` contains cookies
- **Parallel workers** → each worker loads same session file

---

## Rules

- ✅ **Use only in global-setup.ts** — never in test bodies
- ✅ **Create once per test run** — authentication happens before all tests
- ✅ **Save storage state** → `context.storageState({ path: ... })`
- ✅ **Close context and browser** → cleanup after saving session
- ❌ **Don't call from tests** → tests use saved session automatically
- ❌ **Don't use for API calls** → use `SalesforceConnection` for REST API

---

## Complete Example

**global-setup.ts**:
```typescript
import { chromium } from '@playwright/test';
import { SalesforceLoginService, SessionManager } from 'playwright-custom-core';

export default async function globalSetup() {
  console.log('🔐 Authenticating to Salesforce...');
  
  SessionManager.ensureDir();
  
  const browser = await chromium.launch({ headless: true });
  const loginService = new SalesforceLoginService();
  
  try {
    const context = await loginService.login(browser);
    await context.storageState({ path: SessionManager.getStorageStatePath() });
    console.log('✅ Authentication successful');
    
    await context.close();
  } catch (error) {
    console.error('❌ Authentication failed:', error);
    throw error;
  } finally {
    await browser.close();
  }
}
```

**playwright.config.ts**:
```typescript
import { defineConfig } from '@playwright/test';
import { SessionManager } from 'playwright-custom-core';

export default defineConfig({
  globalSetup: './global-setup.ts',
  use: {
    baseURL: process.env.BASE_URL,
    storageState: SessionManager.getStorageStatePath(),
  },
});
```

**Test automatically uses session**:
```typescript
import { test } from '@playwright/test';

test('navigate to accounts', async ({ page }) => {
  await page.goto('/lightning/o/Account/list');
  // Already authenticated — no manual login needed
});
```

---

## Related Utilities
- **SalesforceConnection** — JWT authentication for API calls
- **SessionManager** — Manages storage state file path
- **SalesforceSessionRefreshMiddleware** — Refreshes expired sessions during tests
