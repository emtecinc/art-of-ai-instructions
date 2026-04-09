# SessionManager Utility (INCOMPLETE)

## Purpose
Static utility for managing Playwright storage state file (`.auth/salesforce.json`). Handles authentication directory creation, file existence checks, and cleanup.

---

## When to Use
- In `global-setup.ts` to manage storage state file
- In `global-teardown.ts` to clean up auth files
- In `playwright.config.ts` to reference storage state path
- Check if session file exists before re-authentication

---

## When NOT to Use
- ❌ Inside test bodies → tests use existing session automatically
- ❌ For session refresh → use `SalesforceSessionRefreshMiddleware`
- ❌ For API authentication → use `SalesforceConnection`

---

## Import
```typescript
import { SessionManager } from 'playwright-custom-core';
```

---

## Storage State Location

- **Path**: `<project-root>/.auth/salesforce.json`
- **Lock file**: `<project-root>/.auth/salesforce.json.lock`
- **Directory**: `<project-root>/.auth/`

---

## Methods

### `getStorageStatePath(): string`
Get absolute path to storage state file.

```typescript
const path = SessionManager.getStorageStatePath();
// Returns: '/home/user/project/.auth/salesforce.json'
```

### `getAuthDir(): string`
Get absolute path to auth directory.

```typescript
const dir = SessionManager.getAuthDir();
// Returns: '/home/user/project/.auth'
```

### `getLockPath(): string`
Get absolute path to lock file.

```typescript
const lockPath = SessionManager.getLockPath();
// Returns: '/home/user/project/.auth/salesforce.json.lock'
```

### `ensureDir(): void`
Create `.auth/` directory if it doesn't exist.

```typescript
SessionManager.ensureDir();
// Creates .auth/ directory (no error if already exists)
```

### `exists(): boolean`
Check if storage state file exists.

```typescript
if (SessionManager.exists()) {
  console.log('Session file found');
} else {
  console.log('Session file missing — authentication required');
}
```

### `getMtime(): number`
Get last modification timestamp of storage state file in milliseconds.

```typescript
const lastModified = SessionManager.getMtime();
// Returns: 1710345678901 (timestamp) or 0 if file doesn't exist
```

### `isFresherThan(timestamp: number): boolean`
Check if storage state was modified after given timestamp.

```typescript
const startTime = Date.now();
// ... authentication happens ...
if (SessionManager.isFresherThan(startTime)) {
  console.log('Session was refreshed');
}
```

### `clean(): void`
Delete storage state file and lock file.

```typescript
SessionManager.clean();
// Removes .auth/salesforce.json and .auth/salesforce.json.lock
```

---

## Usage Patterns

### In global-setup.ts
```typescript
import { chromium } from '@playwright/test';
import { SalesforceLoginService, SessionManager } from 'playwright-custom-core';

export default async function globalSetup() {
  SessionManager.ensureDir(); // Create .auth/ directory
  
  const browser = await chromium.launch();
  const loginService = new SalesforceLoginService();
  
  const context = await loginService.login(browser);
  
  // Save authenticated session
  await context.storageState({ path: SessionManager.getStorageStatePath() });
  
  await context.close();
  await browser.close();
}
```

### In playwright.config.ts
```typescript
import { defineConfig } from '@playwright/test';
import { SessionManager } from 'playwright-custom-core';

export default defineConfig({
  use: {
    storageState: SessionManager.getStorageStatePath(),
    // Tests automatically use the saved session
  },
});
```

### In global-teardown.ts
```typescript
import { SessionManager } from 'playwright-custom-core';

export default async function globalTeardown() {
  SessionManager.clean(); // Delete session files
}
```

### Conditional Authentication
```typescript
export default async function globalSetup() {
  SessionManager.ensureDir();
  
  if (SessionManager.exists()) {
    console.log('Existing session found — skipping authentication');
    return;
  }
  
  console.log('No session found — authenticating...');
  const browser = await chromium.launch();
  const loginService = new SalesforceLoginService();
  const context = await loginService.login(browser);
  await context.storageState({ path: SessionManager.getStorageStatePath() });
  await context.close();
  await browser.close();
}
```

---

## Storage State Structure

The `salesforce.json` file contains browser cookies and local storage:

```json
{
  "cookies": [
    {
      "name": "sid",
      "value": "00D...",
      "domain": ".salesforce.com",
      "path": "/",
      "expires": 1710345678,
      "httpOnly": true,
      "secure": true,
      "sameSite": "Lax"
    }
  ],
  "origins": []
}
```

---

## Cross-Process Safety

SessionManager works with `SalesforceSessionRefreshMiddleware` for multi-worker safety:
- `SalesforceSessionRefreshMiddleware` uses file locking (`proper-lockfile`)
- Only one worker refreshes session at a time
- Other workers wait and reload the updated session file
- `getMtime()` and `isFresherThan()` enable freshness checks

---

## Rules

- ✅ **Use in global-setup.ts only** — never inside test bodies
- ✅ **Call ensureDir() first** — ensure `.auth/` exists before creating files
- ✅ **Reference in playwright.config.ts** — `storageState: SessionManager.getStorageStatePath()`
- ✅ **Clean in global-teardown.ts** — remove auth files after test run
- ❌ **Don't call from tests** — tests use session automatically via config
- ❌ **Don't use for API auth** — use `SalesforceConnection` for API operations

---

## Complete Example

**global-setup.ts**:
```typescript
import { chromium } from '@playwright/test';
import { SalesforceLoginService, SessionManager } from 'playwright-custom-core';

export default async function globalSetup() {
  SessionManager.ensureDir();
  
  const browser = await chromium.launch();
  const loginService = new SalesforceLoginService();
  const context = await loginService.login(browser);
  
  await context.storageState({ path: SessionManager.getStorageStatePath() });
  
  await context.close();
  await browser.close();
}
```

**playwright.config.ts**:
```typescript
import { defineConfig } from '@playwright/test';
import { SessionManager } from 'playwright-custom-core';

export default defineConfig({
  globalSetup: './global-setup.ts',
  use: {
    storageState: SessionManager.getStorageStatePath(),
  },
});
```

**global-teardown.ts**:
```typescript
import { SessionManager } from 'playwright-custom-core';

export default async function globalTeardown() {
  SessionManager.clean();
}
```

---

## Related Utilities
- **SalesforceLoginService** — Creates authenticated browser context
- **SalesforceSessionRefreshMiddleware** — Auto-refreshes expired sessions mid-test
- **SalesforceConnection** — API authentication (separate from browser sessions)
