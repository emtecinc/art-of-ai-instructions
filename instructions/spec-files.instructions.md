---
applyTo: "tests/**/*.spec.ts"
---

# Spec File Rules and Guidelines
Spec files are purely DECLARATIVE — they orchestrate workflows with test data. ZERO DOM interaction.

## Imports
```typescript
import { test } from '@playwright/test';           // NEVER import expect
import * as path from 'path';
import { CsvReader, TestDataGenerator, SFDataFactory } from 'playwright-custom-core';
// Import COMPONENT_OBJECT_MAP ONLY when inline-created records need unique value lookup
import { COMPONENT_OBJECT_MAP } from '../../data/component-object-mapping';
import { CreateEntityWorkflow } from '../../workflows/<object>/create-entity-workflow';
```

## Zero Locators — Zero Assertions — High-Level Workflow Calls Only
| Spec MUST ALWAYS | Spec MUST NEVER |
|-------------|------------------|
| Import `test` from `@playwright/test` | Import `expect` |
| Call **high-level** workflow methods (`createEntity()`, `verifyEntityCreated()`) | Call granular workflow steps (`clickNewButton()`, `fillField()`, `clickSave()`) |
| Define test data objects | Use `page.locator()`, `getByRole()`, `getByLabel()`, `getByText()` **except inside `addLocatorHandler` setup** |
| Use `test.step()` for grouping | Call `expect()` anywhere |
| Read CSV via `CsvReader` | Contain hardcoded test data values |
| Access `page` for constructors and `addLocatorHandler`; pass `page` to `waitAndRegisterRecordFromUrl()` | Orchestrate individual UI actions — that belongs in workflows |

### Exception: `addLocatorHandler` Setup (MANDATORY)
The **only place** specs may use `page.locator()` / `page.getByRole()` is inside `beforeEach` when registering `addLocatorHandler` for:
1. **Duplicate detection dialog** — closes "Similar Records Exist" dialog
2. **Toast auto-dismiss** — prevents toast overlays from blocking subsequent interactions
These handlers are **boilerplate infrastructure**, not test logic. Copy verbatim from the template below.

## Test Organization and Tagging
- Use `test.describe()` to group related tests
- Test names start with `should` + business oriented description
- Tags MUST be declared ONLY as an array inside the `test()` function. Usage of tags in `test.describe()` or at any other level is STRICTLY FORBIDDEN.
- The tag array MUST begin with the Jira issue key as the first item (e.g., `@JIRA-101`), followed by relevant test type tags (e.g., `@smoke`, `@regression`).
- If no Jira issue key is provided for the test:
  - Retrieve the alphabetic prefix by reading `TEST_EXEC_PROJECT_KEY` environment variable value
  - Use it to hardcode a dummy Jira tag (e.g., `@JIRA-101`)
  - ALWAYS include a clear reminder in the generated code for the tester to confirm and replace the dummy key with the correct Jira issue key once the test is generated
- EVERY test is independently runnable

## Verification Chain: Spec → Workflow → Page (CRITICAL)
Assertions follow a strict delegation chain:
```
Spec calls: workflow.verifyEntityCreated(data)    ← high-level call
  └─ Workflow: this.testStep('Verify entity name', () => this.detailPage.verifyEntityName(data.name))
       └─ Page Object: await expect(heading).toContainText(data.name)    ← expect() lives HERE only
```
- **Spec:** calls workflow verification methods — never calls `expect()` directly
- **Workflow:** delegates to page object verification methods via `this.testStep()` — never imports `expect`
- **Page Object:** contains all `expect()` assertions — the ONLY layer that imports `expect`

## Cleanup Registration — try/catch/finally Pattern (CRITICAL)
Every record created during a test MUST be registered for cleanup. Toast verification + cleanup MUST be wrapped in `try/catch/finally` to guarantee cleanup runs regardless of toast success.

### Canonical Post-Creation Sequence in Specs
```
1. Call workflow create method           (handles save internally)
2. try/catch/finally: toast + cleanup    (toast is best-effort, cleanup in finally)
3. Call workflow verification methods     (assert the record was created correctly)
```

### Decision Rule
| After save... | Use |
|---|---|
| Page redirected to record detail URL | `await dataFactory.waitAndRegisterRecordFromUrl(page, name)` |
| Page did NOT redirect (inline, modal, quick action, embedded) | `await dataFactory.getRecordIdByField(objectApiName, uniqueField, value)` via `COMPONENT_OBJECT_MAP` |
- Read `.github/utilities/sf-data-factory.md` for COMPLETE reference of cleanup and usage patterns.
- Never use both URL extraction and unique value lookup for the same record.
- The `workflow` handles toast; the `spec` handles identity and cleanup.

## Test Data Loading
CSV is loaded once in `beforeEach` — `csvRow` is available to all tests:
```typescript
// In beforeEach:
const csvPath = path.resolve(__dirname, '../../test-data/<object>/<scenario>.csv');
csvRow = CsvReader.readRow<Record<string, string>>(csvPath, 0)!;
// In test:
const TEST_DATA = {
  name: TestDataGenerator.uniqueName(csvRow.namePrefix),
  type: csvRow.type,
  email: TestDataGenerator.uniqueEmail(csvRow.emailPrefix),
};
```

## SFDataFactory Cleanup Rules
- **Every record created during a test MUST be registered for cleanup** — no orphaned records
- **try/catch/finally** for toast verification + cleanup registration — ALL records (primary + inline) in `finally` block
- **Redirected record** → `await dataFactory.waitAndRegisterRecordFromUrl(page, name)` — waits for full URL, then registers
- **Non-redirect record (inline, modal, quick action, embedded)** → destructure `COMPONENT_OBJECT_MAP` for `objectApiName` and `uniqueField`, then call `await dataFactory.getRecordIdByField(objectApiName, uniqueField, value)` — never hardcode object/field names
- **Inline records in `finally`** — when a workflow creates inline records (child accounts, parent accounts, etc.), register them ALL in the same `finally` block as the primary record
- **Never use both approaches for the same record**
- `teardown()` in `afterEach` — never inside the test body
- **Parent-child cleanup**: register child records FIRST, then parent (teardown deletes in registration order)
- **Multi-record tests**: register ALL created records in the `finally` block using the correct approach per record

## addLocatorHandler Rules
| Rule | Requirement |
|------|-------------|
| Registration | `page.addLocatorHandler()` in `beforeEach` — MANDATORY for record-creating tests |
| `noWaitAfter` | Always pass `{ noWaitAfter: true }` |
| Duplicate dialog | Iterate `closeButtons.count()` with `.catch(() => {})` |
| Toast overlay | Auto-dismiss to prevent blocking subsequent interactions |
| No visibility assertion | Never assert `dialog.not.toBeVisible()` after dismissing |

## Timeout Rules
| Complexity | Timeout |
|------------|---------|
| Simple form (≤ 10 fields) | Default (30s) |
| Large form (> 10 fields / comboboxes) | `120_000` |
| Multiple record creation / inline dialogs | `300_000` |

## Helper Utility Calling
Helper utilities from `instructions/helper-utilities.instructions.md` MUST be called and used within `spec` and `workflow` files, as instructed or as required by the task.

## Complete Spec Template
```typescript
import { test } from '@playwright/test';
import * as path from 'path';
import { CsvReader, TestDataGenerator, SFDataFactory } from 'playwright-custom-core';
// Import COMPONENT_OBJECT_MAP ONLY if inline-created records need unique value lookup
// import { COMPONENT_OBJECT_MAP } from '../../data/component-object-mapping';
import { CreateEntityWorkflow } from '../../workflows/<object>/create-entity-workflow';

test.describe('Entity Creation - Scenario Name', () => {
  let workflow: CreateEntityWorkflow;
  let dataFactory: SFDataFactory;
  let csvRow: Record<string, string>;

  test.beforeEach(async ({ page }) => {
    test.setTimeout(120_000);

    const baseUrl = process.env.BASE_URL || 'https://your-instance.lightning.force.com';
    workflow = new CreateEntityWorkflow(page, baseUrl);
    dataFactory = new SFDataFactory();
    await dataFactory.authenticate();

    // 1. Load test data from CSV
    const csvPath = path.resolve(__dirname, '../../test-data/<object>/<scenario>.csv');
    csvRow = CsvReader.readRow<Record<string, string>>(csvPath, 0)!;

    // 2. Navigate to Salesforce home
    await workflow.navigateToBaseUrl();

    // 3. Close all open tabs for a clean workspace
    await workflow.closeAllPrimaryTabs();

    // 4. Register duplicate detection handler (MANDATORY for record-creating tests)
    await page.addLocatorHandler(
      page.getByRole('dialog', { name: 'Similar Records Exist' }),
      async () => {
        const closeButtons = page.getByRole('dialog', { name: 'Similar Records Exist' })
          .getByRole('button', { name: /Close/i });
        const count = await closeButtons.count();
        for (let i = 0; i < count; i++) {
          await closeButtons.nth(i).click({ timeout: 5_000 }).catch(() => {});
        }
      },
      { noWaitAfter: true }
    );

    // 5. Register toast auto-dismiss handler (MANDATORY — prevents overlay blocking)
    await page.addLocatorHandler(
      // Locates any toast message using locator strategy
      // Clicks the close button on any toast message to prevent overlay issues
      { noWaitAfter: true }
    );
  });

  test.afterEach(async () => {
    await dataFactory.teardown();
  });

  // Use the correct Jira key in the tag below. If key is not provided, generate a key reading the TEST_EXEC_PROJECT_KEY env variable and include a reminder to update it.
  test('should create entity with required fields', { 
      tag: ['@JIRA-101', '@smoke'] 
    }, async ({ page }) => {
    const entityData = {
      name: TestDataGenerator.uniqueName(csvRow.namePrefix),
      type: csvRow.type,
    };

    // Act — workflow handles save internally (no toast inside create)
    await test.step('Create entity', async () => {
      await workflow.createEntity(entityData);
    });

    // Toast verification + cleanup — try/catch/finally guarantees cleanup runs
    // ALL records (primary + inline) registered in finally — never in separate steps
    await test.step('Verify toast and register cleanup', async () => {
      let toastError: unknown;
      try {
        await workflow.verifySuccessToast(entityData.name);
      } catch (error) {
        toastError = error;
      } finally {
        // Primary record: wait for redirect URL to resolve, then register
        await dataFactory.waitAndRegisterRecordFromUrl(page, entityData.name);
        // Inline-created records: use COMPONENT_OBJECT_MAP — never hardcode objectApiName/uniqueField
        // const { objectApiName, uniqueField } = COMPONENT_OBJECT_MAP['Account'];
        // await dataFactory.getRecordIdByField(objectApiName, uniqueField, entityData.childAccountName);
      }
      if (toastError) throw toastError;
    });

    // Assert — workflow handles all verifications internally
    await test.step('Verify entity created', async () => {
      await workflow.verifyEntityCreated(entityData);
    });
  });
});
```