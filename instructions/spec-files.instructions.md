---
applyTo: "tests/**/*.spec.ts"
---

# Spec File Rules

## Spec files are purely declarative — they orchestrate workflows with test data. ZERO DOM interaction.

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

| ✅ Spec CAN | ❌ Spec MUST NOT |
|-------------|------------------|
| Import `test` from `@playwright/test` | Import `expect` |
| Call **high-level** workflow methods (`createEntity()`, `verifyEntityCreated()`) | Call granular workflow steps (`clickNewButton()`, `fillField()`, `clickSave()`) |
| Define test data objects | Use `page.locator()`, `getByRole()`, `getByLabel()`, `getByText()` |
| Use `test.step()` for grouping | Call `expect()` anywhere |
| Read CSV via `CsvReader` | Contain hardcoded test data values |
| Access `page` for constructors and `page.url()` | Orchestrate individual UI actions — that belongs in workflows |

## Verification Chain: Spec → Workflow → Page (CRITICAL)

Assertions follow a strict delegation chain:

```
Spec calls: workflow.verifyEntityCreated(data)    ← high-level call
  └─ Workflow: this.step('Verify entity name', () => this.detailPage.verifyEntityName(data.name))
       └─ Page Object: await expect(heading).toContainText(data.name)    ← expect() lives HERE only
```

- **Spec:** calls workflow verification methods — never calls `expect()` directly
- **Workflow:** delegates to page object verification methods via `this.step()` — never imports `expect`
- **Page Object:** contains all `expect()` assertions — the ONLY layer that imports `expect`

## Cleanup Registration — All Creation Patterns (CRITICAL)

Every record created during a test MUST be registered for cleanup. Cleanup registration is **non-negotiable** whenever a record was created and enough identity is available to register it.

### Canonical Post-Creation Sequence in Specs

After a workflow creates a record, the spec MUST follow this sequence:

```
1. Call workflow create method           (handles save + toast verification internally)
2. Determine record identity             (using the correct pattern for the creation flow)
3. Register cleanup                      (mandatory — MUST NOT depend on toast success)
4. Call workflow verification methods     (assert the record was created correctly)
```

**Key rules:**
- Cleanup registration MUST NOT depend on toast verification success — toast is best-effort
- Cleanup registration MUST NOT assume URL redirect is the only identity pattern
- Cleanup MUST happen before verification steps (so cleanup runs even if verification fails)

### Record Identity Patterns

| Creation Flow | Identity Source | Cleanup Registration |
|---|---|---|
| Full-page create (form → Save → redirect) | `page.url()` after redirect | `dataFactory.registerRecordFromUrl(page.url(), name)` |
| Inline/related record (no redirect) | Unique field value (already known from test data) | `await dataFactory.getRecordIdByField(objectApiName, field, value)` |
| Modal create (dialog → Save → close) | `page.url()` if redirected, else unique field query | Use whichever identity source is available |
| Quick action / embedded / LWC | Unique field value or captured reference | `await dataFactory.getRecordIdByField(objectApiName, field, value)` |

### Decision Rule

| After save... | Use |
|---|---|
| Page redirected to record detail URL | **URL Extraction** — `dataFactory.registerRecordFromUrl(page.url(), name)` |
| Page did NOT redirect (inline, modal, quick action, embedded) | **Unique Value Lookup** — `await dataFactory.getRecordIdByField(objectApiName, field, value)` |
| Record reference already captured in flow | Use the captured reference directly |

- See `utility/sf-data-factory.md` for full API reference and usage patterns.
- Never use both URL extraction and unique value lookup for the same record.
- The workflow determines toast handling; the **spec** determines identity and cleanup — see `workflows.instructions.md`.

## Complete Spec Template

```typescript
import { test } from '@playwright/test';
import * as path from 'path';
import { CsvReader, TestDataGenerator, SFDataFactory } from 'playwright-custom-core';
// Import COMPONENT_OBJECT_MAP ONLY if inline-created records need unique value lookup
// import { COMPONENT_OBJECT_MAP } from '../../data/component-object-mapping';
import { CreateEntityWorkflow } from '../../workflows/<object>/create-entity-workflow';

test.describe('Entity Creation - Scenario Name @smoke', () => {
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
  });

  test.afterEach(async () => {
    await dataFactory.teardown();
  });

  test('should create entity with required fields @smoke', async ({ page }) => {
    const entityData = {
      name: TestDataGenerator.uniqueName(csvRow.namePrefix),
      type: csvRow.type,
    };

    // Act — workflow handles save + toast verification internally
    await test.step('Create entity', async () => {
      await workflow.createEntity(entityData);
    });

    // Cleanup — determine identity and register (mandatory, before verification)
    await test.step('Register cleanup', async () => {
      // For redirect-based creation: extract from URL
      dataFactory.registerRecordFromUrl(page.url(), entityData.name);
      // For non-redirect creation: query by unique field instead
      // await dataFactory.getRecordIdByField('Entity__c', 'Name', entityData.name);
    });

    // Assert — workflow handles all verifications internally
    await test.step('Verify entity created', async () => {
      await workflow.verifyEntityCreated(entityData);
    });
  });
});
```

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
- **Cleanup registration is non-negotiable** when a record was created and identity is available
- **Cleanup MUST NOT depend on toast success** — toast is best-effort (see `salesforce-stability.instructions.md`)
- **Redirect URL is only ONE identity pattern** — use the correct pattern for the creation flow (see table above)
- **Redirected record** → `registerRecordFromUrl(page.url(), name)` — extract recordId from URL after save
- **Non-redirect record (inline, modal, quick action, embedded)** → `await getRecordIdByField(objectApiName, uniqueField, value)` — query by unique value
- **Never use both approaches for the same record**
- `teardown()` in `afterEach` — never inside the test body
- **Parent-child cleanup**: register child records FIRST, then parent (teardown deletes in registration order)
- **Multi-record tests**: register each created record using the correct approach per record

```typescript
// Redirected record — extract from URL after save
dataFactory.registerRecordFromUrl(page.url(), entityData.name);

// Non-redirect record (inline, modal, quick action, embedded) — query by unique value
await dataFactory.getRecordIdByField(childObjectApiName, childUniqueField, childValue);
```

## Timeout Rules

| Complexity | Timeout |
|------------|---------|
| Simple form (≤ 10 fields) | Default (30s) |
| Large form (> 10 fields / comboboxes) | `120_000` |
| Multiple record creation / inline dialogs | `300_000` |

## Test Organization

- `test.describe()` to group related tests
- Test names start with `should` + business description
- Tag with `@smoke` or `@regression`
- Each test is independently runnable

## Helper-utility calling

- Helper utilities mentioned in `instructions/helper-utilities.instructions.md` are to be called and used in spec files, as per instructions or demand.
