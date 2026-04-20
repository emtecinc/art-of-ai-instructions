# SFDataFactory Utility 

## When to Use
- **Every test that creates Salesforce records** (UI or API) must use SFDataFactory
- Register records for cleanup immediately after creation
- Automatic deletion in `test.afterEach()` regardless of pass/fail
- Create/update/fetch/delete records via REST API
- Batch operations via Composite API (up to 25 subrequests)
- Query record ID by unique field value after UI creation

---

## When NOT to Use
- ❌ For read-only tests that don't create data
- ❌ Inside test body for teardown → use `test.afterEach()`

---

## Import
```typescript
import { SFDataFactory } from 'playwright-custom-core';
import { COMPONENT_OBJECT_MAP } from '../../data/component-object-mapping';
```

---

## Core Concepts

### Unique-Field Approach (UI Tests)
- Only **one field per object** must be unique per test run
- All other fields can remain static from CSV
- Use `COMPONENT_OBJECT_MAP` to identify `objectApiName` and `uniqueField`

### Parent-Child Deletion Order
- Register **child records first**, then parent records
- `teardown()` deletes in registration order → child deleted before parent
- Prevents foreign-key constraint errors

---

## Quick Start Patterns

### Pattern 1: UI Record Creation + Cleanup
```typescript
import { SFDataFactory, CsvReader, TestDataGenerator } from 'playwright-custom-core';
import { COMPONENT_OBJECT_MAP } from '../../data/component-object-mapping';

let dataFactory: SFDataFactory;

test.beforeEach(async ({ page }) => {
  dataFactory = new SFDataFactory();
  await dataFactory.authenticate();
  // ... other setup
});

test.afterEach(async () => {
  await dataFactory.teardown(); // ALWAYS runs — pass or fail
});

// Pattern 1.1: Use when a related record is created inline without redirecting
test.step('Register inline-created account for cleanup', async () => {
  const { objectApiName, uniqueField } = COMPONENT_OBJECT_MAP['Account'];
  await dataFactory.getRecordIdByField(objectApiName, uniqueField, accountName);
});

// Pattern 1.2: Use when Save redirects to the created record's detail page
await test.step('Verify toast and register cleanup', async () => {
  let toastError: unknown;
  try {
    await workflow.verifySuccessToast(data.name);
  } catch (error) {
    toastError = error;
  } finally {
    // Extract recordId from URL — save redirected to record page
    dataFactory.registerRecordFromUrl(page.url(), data.name);
  }

  if (toastError) throw toastError;
});
```

### Pattern 2: API Record Creation + Cleanup
```typescript
import { test } from '@playwright/test';
import { SFDataFactory, PayloadBuilder, TestDataGenerator } from 'playwright-custom-core';

let dataFactory: SFDataFactory;

test.beforeEach(async () => {
  dataFactory = new SFDataFactory();
  await dataFactory.authenticate();
});

test.afterEach(async () => {
  await dataFactory.teardown();
});

test('should create account via API', async () => {
  const payload = PayloadBuilder...
  // payload building

  const accountId = await dataFactory.createRecord('Account', payload);
  // Auto-registered for cleanup
  
  const account = await dataFactory.fetchRecord('Account', accountId, ['Name', 'Type']);
  expect(account.Name).toContain('APIAccount_'); // used TestDataGenerator.uniqueName('APIAccount') in payload builder
});
```

### Pattern 3: Composite API — Parent + Child
```typescript
test('should create account and contacts via Composite API', async () => {
  accountPayload = PayloadBuilder...
  contact1Payload = PayloadBuilder...
  contact2Payload = PayloadBuilder...
  // AccountId will be set via cross-reference in subrequest body

  const results = await dataFactory.executeCompositeRequest([
    {
      method: 'POST',
      url: '/services/data/v65.0/sobjects/Account',
      referenceId: 'refAccount',
      body: accountPayload,
    },
    {
      method: 'POST',
      url: '/services/data/v65.0/sobjects/Contact',
      referenceId: 'refContact1',
      body: contact1Payload,
    },
    {
      method: 'POST',
      url: '/services/data/v65.0/sobjects/Contact',
      referenceId: 'refContact2',
      body: contact2Payload,
    },
  ]);

  // Extract IDs and register for cleanup (child-before-parent order)
  const accountId = results[0].body!.id as string;
  const contact1Id = results[1].body!.id as string;
  const contact2Id = results[2].body!.id as string;

  dataFactory.registerForCleanup('Contact', contact1Id, contact1Name);
  dataFactory.registerForCleanup('Contact', contact2Id, contact2Name);
  dataFactory.registerForCleanup('Account', accountId, accountName);
  
  // Verify
  const account = await dataFactory.fetchRecord('Account', accountId);
  expect(account.Name).toBe(accountName);
});
```

### Record CRUD

**Parameters**:
- `sObject`: Object API name (`'Account'`, `'Contact'`, `'CustomObject__c'`)
- `payload`: Record fields as plain object or from `PayloadBuilder`
- `autoRegister`: Auto-register for cleanup (default: `true`)
- `recordId`: 15 or 18 character Salesforce ID
- `fields`: Array of field API names (omit for `FIELDS(ALL)`)
- `name`: Optional display name for logging

### Record Creation

#### `createRecord(sObject, payload, autoRegister?): Promise<string>`
Create a record via REST API and auto-register for cleanup.

**Features**:
- Bypasses Salesforce duplicate rules via `Sforce-Duplicate-Rule-Header: allowSave=true`
- Auto-registers for cleanup unless disabled

---

### Record Update

#### `updateRecord(sObject, recordId, payload): Promise<void>`
Update an existing record via PATCH (partial update).

---

### Record Fetch

#### `fetchRecord<T>(sObject, recordId, fields?): Promise<T>`
Fetch a single record via SOQL Query API.

---

### Record Delete

#### `deleteRecord(sObject, recordId, name?): Promise<void>`
Delete a single record immediately via REST API.

**Note**: Record remains in cleanup registry — `teardown()` handles 404 gracefully.

---

### UI Record Registration

#### `getRecordIdByField(sObject, fieldName, value, autoRegister?): Promise<string>`
Query Salesforce for record ID by unique field value and auto-register for cleanup.

**Parameters**:
- `sObject`: Object API name
- `fieldName`: Field to search by (`'Name'`, `'Email'`, `'SerialNumber__c'`)
- `value`: Unique value created by `TestDataGenerator`
- `autoRegister`: Auto-register for cleanup (default: `true`)

---

### Manual Registration

#### `registerForCleanup(sObject, recordId, name?): void`
Manually register a record for cleanup (used when `autoRegister: false`).

---

### Composite API

#### `executeCompositeRequest(subrequests, allOrNone?): Promise<CompositeSubresponse[]>`
Execute up to 25 subrequests in a single HTTP round-trip.

**Parameters**:
- `subrequests`: Array of `CompositeSubrequest` (POST/PATCH/DELETE/GET)
- `allOrNone`: Rollback all on any failure (default: `false`)

**Returns**: Ordered array of `CompositeSubresponse` matching input order

**Features**:
- Cross-reference earlier results: `@{referenceId.field}`
- Max 25 subrequests per call
- Preserves order — `results[i]` corresponds to `subrequests[i]`
- Bypasses duplicate rules

---

### Utility Methods

#### `getCleanupCount(): number`
Get count of records pending cleanup.

#### `getRegisteredRecords(): ReadonlyArray<{ type, id, name? }>`
Get snapshot of registered records (read-only).

#### `generateUniqueName(baseName): string`
Generate unique name (legacy — use `TestDataGenerator.uniqueName()` instead).

---

## Composite API Types

### `CompositeSubrequest`
```typescript
interface CompositeSubrequest {
  method: 'POST' | 'PATCH' | 'DELETE' | 'GET';
  url: string;                    // Relative SF REST URL
  referenceId: string;            // Alphanumeric ID for cross-reference
  body?: Record<string, unknown>; // Payload (omit for DELETE/GET)
}
```

### `CompositeSubresponse`
```typescript
interface CompositeSubresponse {
  body: Record<string, unknown> | null; // Response body (null for 204)
  httpStatusCode: number;               // HTTP status for this subrequest
  referenceId: string;                  // Matches request referenceId
  httpHeaders: Record<string, string>;  // Response headers
}
```

---

## Critical Rules

- ✅ **Only uniqueField must be unique** — all other fields stay static from CSV
- ✅ **Register BEFORE assertions** — cleanup step comes immediately after save
- ✅ **Always use test.afterEach()** — never call `teardown()` inside test body
- ✅ **Child-before-parent order** — register child first when creating related records
- ✅ **Query via getRecordIdByField** — never extract ID from toast message or URL
- ✅ **ALWAYS use PayloadBuilder for API creation** — structured, type-safe payloads
- ✅ **ALWAYS use `executeCompositeRequest()` for more than 1 read/write requests** — avoid using `fetchRecord`, `deleteRecord`, `updateRecord` and `createRecord` for multiple API calls unless necessary. Leverage references between subrequests for parent-child relationships and to ensure atomicity when needed.
- ✅ **API methods should be called in workflow** — ensure API methods are invoked within the appropriate test workflow
- ❌ **Never register during impersonation** — call `ImpersonationHelper.logBack()` first
- ❌ **Don't exceed 25 subrequests** — Composite API limit

---