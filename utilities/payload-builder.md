# PayloadBuilder Utility

## When to Use
- Creating Salesforce records in bulk via REST API (`SFDataFactory.executeCompositeRequest`)
- Creating Salesforce records individually (`SFDataFactory.createRecord` and `SFDataFactory.updateRecord`)
- Build simple or complex payloads with nested objects
- Merge static template/CSV data with dynamic test values as required

---

## When NOT to Use
- ❌ For UI-based record creation → use Page Objects + Workflows
- ❌ For read-only API operations → no payload needed

---

## Import
```typescript
import { PayloadBuilder } from 'playwright-custom-core';
```

---

## Core Concepts

### Payload building patterns:
1. Can build from empty object.
2. Can build from existing object (e.g. data in a CSV row/file). **Preferred** 

---

## Quick Start

### Pattern 1: Build from Empty + Add Fields
```typescript
const payload = PayloadBuilder
  .empty<AccountPayload>()
  .set('Name', TestDataGenerator.uniqueName('Account'))
  .set('Type', 'Customer')
  .set('Industry', 'Technology')
  .build();

// Result: { Name: "Account_1710345678901", Type: "Customer", Industry: "Technology" }
```

### Pattern 2: Read from CSV + Override Fields
```typescript
import { PayloadBuilder, TestDataGenerator } from 'playwright-custom-core';

// ex. Read test-data/account/account.csv
const csvPath = path.resolve(__dirname, '../../test-data/account/account.csv');
const csvRow = CsvReader.readRow<Record<string, string>>(csvPath, 0)!;

const AccountPayload = {
  Name: csvRow.name,          // Static value from CSV
  Type: csvRow.type,          // Static value from CSV
  Industry: 'Finance',        // Dynamic override in test code
  BillingAddress: {           // Nested object with dynamic overrides
    City: 'San Francisco',
    State: 'CA',
  },
};
const payload = PayloadBuilder
  .fromObject(AccountPayload)
  .set('Name', TestDataGenerator.uniqueName('Account'))
  .set('Industry', 'Finance')
  .build();

// Result: { Name: "Account_1710345678901", Type: "Customer", Industry: "Finance" }
```

### Pattern 4: Merge Multiple Fields
```typescript
const dynamicFields = {
  Name: TestDataGenerator.uniqueName('Account'),
  Phone: TestDataGenerator.randomPhone(),
  Website: 'https://example.com',
};

const payload = PayloadBuilder
  // fromObject or empty or fromFile
  .merge(dynamicFields)
  .build();
```

---

### `fromObject<T>(base: T): PayloadBuilder<T>`
Create builder from existing object.

**Parameters**:
- `base`: Base object to use as starting payload

```typescript
const basePayload = { Name: 'Test', Type: 'Customer' }; // type safe object by reading from CSV or defined inline
const builder = PayloadBuilder.fromObject(basePayload);
```

---

### `empty<T>(): PayloadBuilder<T>`
Create empty builder.

```typescript
const builder = PayloadBuilder.empty<AccountPayload>();
```

---

## Configuration

### `strict(): PayloadBuilder<T>`
Enable strict mode — only allows setting keys that exist in base payload.

**Returns**: `this` (for chaining)

```typescript
const payload = PayloadBuilder
  // fromObject or empty
  .strict()
  .set('Name', 'Valid Field')      // ✅ OK — exists in template
  .set('InvalidField', 'Value')    // ❌ ERROR — not in template
  .build();
```

---

## Field Setters

### `set<K>(key: K, value: T[K] | null | undefined): PayloadBuilder<T>`
Set a single top-level field value.

---

### `setNested(keyPath: string, value: unknown): PayloadBuilder<T>`
Set nested field using dot-notation path.

---

### `merge(overrides: Partial<T>): PayloadBuilder<T>`
Merge multiple field overrides at once.

---

### `remove<K>(key: K): PayloadBuilder<T>`
Remove a field from the payload.

**Parameters**:
- `key`: Field name to remove

**Returns**: `this` (for chaining)

```typescript
builder
  .set('Name', 'Account')
  .set('TempField', 'Temporary')
  .remove('TempField');  // TempField not in final payload
```

---

### `toJSON(pretty?: boolean): string`
Build and return payload as JSON string.

---

### `peek(): Readonly<T>`
Peek at current payload state without building (for debugging).

---

## Usage with SFDataFactory

### Complete Example — Create multiple Accounts
```typescript
import { test } from '@playwright/test';
import { SFDataFactory, PayloadBuilder, TestDataGenerator } from 'playwright-custom-core';

let dataFactory: SFDataFactory;

test.beforeEach(async () => {
  // data factory and other initialization
});

test.afterEach(async () => {
  // data factory teardown
});

test('should create account with address via API', async () => {
  const accountPayload1 = PayloadBuilder
    .// fromObject or empty or fromFile
    .set('Name', TestDataGenerator.uniqueName('APIAccount'))
    .set('Type', 'Customer')
    .setNested('BillingAddress.City', 'San Francisco')
    .setNested('BillingAddress.State', 'CA')
    .build();

  const accountPayload2 = PayloadBuilder
    .// fromObject or empty or fromFile
    .set('Name', TestDataGenerator.uniqueName('APIAccount2'))
    .set('Type', 'Partner')
    .build();

  const sub1 = {
    method: 'POST',
    url: '/services/data/v65.0/sobjects/Account',
    referenceId: 'refAccount1',
    body: accountPayload1,
  };
  const sub2 = {
    method: 'POST',
    url: '/services/data/v65.0/sobjects/Account',
    referenceId: 'refAccount2',
    body: accountPayload2,
  };
  const response = await dataFactory.executeCompositeRequest([sub1, sub2]);
  
  // Extract the data for assertions as needed

});
```

### Complete Example — Create Contact with Account
```typescript
test('should create contact linked to account via API', async () => {
  // 1. Create parent account
  const accountPayload = PayloadBuilder
    .// fromObject or empty or fromFile
    .set('Name', TestDataGenerator.uniqueName('ParentAccount'))
    .build();

  // 2. Create child contact
  const contactPayload = PayloadBuilder
    .// fromObject or empty or fromFile
    .set('LastName', TestDataGenerator.uniqueName('Contact'))
    .set('Email', TestDataGenerator.uniqueEmail('contact'))
    .set('AccountId', '@{refAccount.id}')  // Link to parent
    .build();

  const response = await dataFactory.executeCompositeRequest([
    {
      method: 'POST',
      url: '/services/data/v65.0/sobjects/Account',
      referenceId: 'refAccount',
      body: accountPayload
    },
    {
      method: 'POST',
      url: '/services/data/v65.0/sobjects/Contact',
      referenceId: 'refContact',
      body: contactPayload
    }
  ]);

  // Ensure to extract correct ids from response for assertions
  const accountId = response.results[0].id;
  const contactId = response.results[1].id;

  // 3. Verify contact is linked to account
  const contact = await dataFactory.fetchRecord('Contact', contactId, ['AccountId']);
  expect(contact.AccountId).toBe(accountId);

});
```

---

## Advanced Patterns

### Pattern: Conditional Field Updates
```typescript
const accountObj = {
  accountType: csvRow.accountType
};

const builder = PayloadBuilder.fromObject(accountObj)
  .set('Name', TestDataGenerator.uniqueName('Account'));

// Conditionally add fields
if (accountObj.accountType === 'Enterprise') {
  builder.set('Industry', 'Technology');
  builder.setNested('BillingAddress.Country', 'USA');
}

const payload = builder.build();
```

### Pattern: Reuse Builder with Multiple Overrides
```typescript
const baseBuilder = PayloadBuilder.fromObject({
  FirstName: 'Test',
  LastName: 'Contact',
  Email: 'test@example.com'
});

// Create multiple contacts with same template
const contact1 = baseBuilder
  .set('LastName', TestDataGenerator.uniqueName('Contact1'))
  .set('Email', TestDataGenerator.uniqueEmail('contact1'))
  .build();

const contact2 = baseBuilder
  .set('LastName', TestDataGenerator.uniqueName('Contact2'))
  .set('Email', TestDataGenerator.uniqueEmail('contact2'))
  .build();
```
---

## Best Practices

- ✅ **Read data from related CSV** — use to build with `fromObject` by refering to any relevant CSV file and row for the test case
- ✅ **Confirm types before building payload** — use TypeScript interfaces and `strict()` to catch errors early. ex. Date in CSV is string, but API expects Date in ISO format → convert in test code before building payload 
- ✅ **Override only unique/dynamic fields** — leverage static values from CSV and only override fields that need to be unique or dynamic for the test case.
- ✅ **Build with minimum payload** — only include required fields and those relevant to the test case or any specified.
- ✅ **Combine with `TestDataGenerator`** — generate unique values for required fields
- ✅ **Use `peek()` for debugging ONLY** — inspect intermediate state
- ❌ **Don't hardcode field values in test code** — use CSV or templates
- ❌ **Don't mutate original templates** — builder creates deep clones
- ❌ **Don't use for simple payloads** — plain objects sufficient for ≤3 fields

---

## Rules

- ✅ **Always load from object** — don't create builders manually unless no CSV data is available
- ✅ **Use chaining for multiple operations** — improves readability
- ✅ **Call `build()` last** — returns final payload ready for API
- ❌ **Don't modify returned payload** — it's a deep clone, safe to mutate if needed
- ❌ **Don't cache builders across tests** — create fresh for each test

---
