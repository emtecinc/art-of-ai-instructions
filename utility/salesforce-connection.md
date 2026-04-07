# SalesforceConnection Utility

## Purpose
Singleton for Salesforce OAuth 2.0 JWT Bearer Flow authentication. Manages access tokens and instance URLs for all Salesforce API operations.

---

## When to Use
- Access Salesforce REST API, Tooling API, or Bulk API
- Authenticate utilities (SFDataFactory, batch services, email providers)
- Share credentials across all framework components
- Obtain org URL for navigation

---

## When NOT to Use
- ❌ Inside test bodies → use via other utilities (SFDataFactory, batch executors)
- ❌ For UI browser login → use `SalesforceLoginService`
- ❌ Creating multiple instances → always use singleton `getInstance()`

---

## Import
```typescript
import { SalesforceConnection } from 'playwright-custom-core';
```

---

## Required Environment Variables

Set in `.env` or CI/CD:

| Variable | Description | Default |
|---|---|---|
| `SF_USERNAME` | Automation user's username | *(required)* |
| `SF_CLIENT_ID` | Connected App consumer key | *(required)* |
| `SF_PRIVATE_KEY_PATH` | Path to RSA private key (`.key`/`.pem`) | `./server.key` |
| `SF_LOGIN_URL` | OAuth login URL | `https://login.salesforce.com` |
| `BASE_URL` | Salesforce org base URL | — |

---

## Singleton Pattern

**Always use `getInstance()`** — never construct directly.

```typescript
const conn = SalesforceConnection.getInstance();
```

---

## Methods

### `getConnection(): Promise<SalesforceCredentials>`
Authenticate and return cached credentials.

**Returns**: `{ accessToken: string, instanceUrl: string }`

```typescript
const { accessToken, instanceUrl } = await conn.getConnection();
// First call: performs JWT authentication
// Subsequent calls: returns cached credentials
```
---

## Rules

- ✅ **Always use singleton** — `getInstance()`, never `new SalesforceConnection()`
- ✅ **Call once, cache everywhere** — first call authenticates, subsequent calls use cache
- ✅ **Use in utilities, not tests** — SFDataFactory, batch services, email providers
- ✅ **Reset when needed** — `reset()` for fresh JWT, `resetInstance()` between suites
- ❌ **Never construct in test bodies** — use via utilities that depend on it
- ❌ **Never hardcode credentials** — always use environment variables

---

## Complete Example

```typescript
import { SalesforceConnection } from 'playwright-custom-core';

// In a utility class
export class MyService {
  async querySalesforce() {
    const conn = SalesforceConnection.getInstance();
    const { accessToken, instanceUrl } = await conn.getConnection();
    // Pass the connection or promise to utilities as required
  }
}
```
