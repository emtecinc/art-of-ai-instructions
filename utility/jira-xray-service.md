
# Jira-Xray Integration Service (INCOMPLETE)

## Purpose
Integrate Playwright test execution with Jira and Xray Test Management. Fetch test cases from Jira - based on labels or linked to a worktype, run corresponding Playwright specs, generate Xray-compatible results, and upload to Xray Cloud.

---

## When to Use
- Execute tests based on Jira label(s) or linked to a worktype
- Upload Playwright test results to Xray Test Management
- Create or update Xray Test Executions from CI/CD
- Sync test automation with Jira test case management

---

## When NOT to Use
- ❌ For non-Jira test management → use standard Playwright reporting
- ❌ When not using Xray → use Playwright HTML/JSON reporters
- ❌ For local test runs → intended for CI/CD integration

---

## Import
```typescript
import {
  fetchIssuesByLabel,
  fetchByKey,
  fetchIssuesByFilter,
  runTestsWithResultsUpload,
  findSpecFiles,
  buildXrayPayload,
  importResults
} from 'playwright-custom-core';
```

---

## Required Environment Variables

### Jira Configuration
| Variable | Description |
|---|---|
| `JIRA_URL` | Jira base URL (e.g., `https://company.atlassian.net`) |
| `JIRA_EMAIL` | Jira user email |
| `JIRA_TOKEN` | Jira API token |

### Xray Configuration
| Variable | Description |
|---|---|
| `XRAY_CLIENT_ID` | Xray Cloud client ID |
| `XRAY_CLIENT_SECRET` | Xray Cloud client secret |
| `XRAY_BASE_URL` | Xray Cloud API URL (e.g., `https://xray.cloud.getxray.app/api/v2`) |

### Test Execution Configuration
| Variable | Description | Default |
|---|---|---|
| `TEST_EXEC_PROJECT_KEY` | Jira project key (e.g., `SFTEST`) | — |
| `TEST_EXEC_ISSUE_TYPE` | Issue type to filter (e.g., `Test`) | — |
| `TEST_EXEC_EXECUTION_KEY` | Existing execution key to update | *(creates new)* |
| `TEST_EXEC_INPUT_TYPE` | Input mode: `label`, `key` | — |
| `TEST_EXEC_INPUT_VALUE` | Value for input type (label(s), Test Set key or Test Execution key.) | — |
| `TEST_EXEC_JIRA_FIELDS` | Comma-separated Jira fields to retrieve | `summary,status,issuetype` |

---

## Core Workflows

### 1. Fetch Tests by Label(s)
Fetch Jira test cases with specific label(s), run matching Playwright specs, upload results.

**Environment setup**:
```bash
TEST_EXEC_INPUT_TYPE=label
TEST_EXEC_INPUT_VALUE=smoke,regression
TEST_EXEC_PROJECT_KEY=SFTEST
TEST_EXEC_ISSUE_TYPE=Test
```

**Usage**:
```typescript
import { fetchIssuesByLabel } from 'playwright-custom-core/utils/jira-xray';

await fetchIssuesByLabel();
// 1. Searches Jira for issues of type Test with label(s) 'smoke','regression',etc.
// 2. Get their key(s)
// 3. Finds matching spec files
// 4. Runs Playwright tests
// 5. Generates Xray payload
// 6. Uploads results to Xray
```

### 2. Fetch Tests linked to a issue/worktype
Fetch test cases linked to a Test Execution or Test Set, run them, create new or update Execution with results.

**Environment setup**:
```bash
TEST_EXEC_PROJECT_KEY=SFTEST
TEST_EXEC_INPUT_TYPE=key
TEST_EXEC_INPUT_VALUE=SFTEST-123
TEST_EXEC_EXECUTION_KEY=SFTEST-124
```

**Usage**:
```typescript
import { fetchByKey } from 'playwright-custom-core';

await fetchByKey();
// 1. Fetches linked test keys to Test Set/Execution SFTEST-123
// 2. Finds matching spec files
// 3. Runs Playwright tests
// 4. Updates SFTEST-124 with results or creates new if .env variable is empty
```
---


## Spec File Mapping

Tests will have 

```typescript
import { test } from '@playwright/test';

test('should create account successfully @SFTEST-456', async ({ page }) => {
  // Test implementation
});

test('should validate required fields @SFTEST-457 @smoke', async ({ page }) => {
  // Test implementation
});
```

**Mapping rules**:
- Test key extracted from `@SFTEST-456` tag format
- `findSpecFiles(['SFTEST-456', 'SFTEST-457'])` → finds matching specs
- Runs only tests with matching `@tag` annotations

---

## Result Upload

### Create New Test Execution
Omit `TEST_EXEC_EXECUTION_KEY` to create new execution:

```bash
TEST_EXEC_PROJECT_KEY=SFTEST
# No TEST_EXEC_EXECUTION_KEY
```

**Result**: Creates new Test Execution in Jira with results.

### Update Existing Test Execution
Provide `TEST_EXEC_EXECUTION_KEY` to update existing execution:

```bash
TEST_EXEC_EXECUTION_KEY=SFTEST-123
TEST_EXEC_PROJECT_KEY=SFTEST
```

**Result**: Updates SFTEST-123 with new test results.

---

## Low-Level APIs

### `findSpecFiles(testKeys: string[]): Promise<string[]>`
Find spec files containing test keys.

```typescript
const files = await findSpecFiles(['SFTEST-456', 'SFTEST-457']);
// Returns: ['tests/account/create-account.spec.ts', 'tests/contact/create-contact.spec.ts']
```

### `buildXrayPayload(options): void`
Generate Xray-compatible JSON from Playwright results.

```typescript
buildXrayPayload({
  testExecutionKey: 'SFTEST-123', // Optional — omit to create new
  projectKey: 'SFTEST',
});
// Reads: playwright-report/results.json
// Writes: xray-results.json
```

### `importResults(config): Promise<string | null>`
Upload results to Xray Cloud.

```typescript
const executionKey = await importResults({
  xray: {
    auth: {
      clientId: process.env.XRAY_CLIENT_ID!,
      clientSecret: process.env.XRAY_CLIENT_SECRET!,
    },
    baseUrl: process.env.XRAY_BASE_URL!,
  },
});
// Returns: 'SFTEST-789' (newly created or updated execution key)
```
---

## Rules

- ✅ **Tag tests with Jira keys** — `@SFTEST-456` format required
- ✅ **Run from CI/CD** — designed for automated pipelines
- ✅ **Set all required env vars** — Jira, Xray, project config
- ✅ **Use label-based for regression** — flexible test selection
- ✅ **Use execution-based for sprints** — run specific execution suites
- ❌ **Don't use for local runs** — overhead of Jira/Xray integration
- ❌ **Don't hardcode credentials** — always use env vars/secrets

---

## Complete Example

**Test file with Jira tags**:
```typescript
// tests/account/create-account-required-fields.spec.ts
import { test, expect } from '@playwright/test';

test('should create account with required fields @SFTEST-456 @smoke', async ({ page }) => {
  // Test implementation
});

test('should validate account name @SFTEST-457 @regression', async ({ page }) => {
  // Test implementation
});
```

**CI/CD execution**:
```bash
export TEST_EXEC_INPUT_TYPE=label
export TEST_EXEC_INPUT_VALUE=smoke
export TEST_EXEC_PROJECT_KEY=SFTEST
export TEST_EXEC_ISSUE_TYPE=Test
export JIRA_URL=https://company.atlassian.net
export JIRA_EMAIL=automation@company.com
export JIRA_TOKEN=xxxxx
export XRAY_CLIENT_ID=xxxxx
export XRAY_CLIENT_SECRET=xxxxx

npm run test:xray
# Fetches tests labeled 'smoke' → runs them → uploads results to Xray
```

---

## Related Utilities
- **Playwright Reporter** — Standard HTML/JSON reports (separate from Xray)
- **CI/CD Pipelines** — GitHub Actions, Jenkins, Azure Pipelines integration
