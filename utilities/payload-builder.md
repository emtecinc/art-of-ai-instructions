# PayloadBuilder Utility

## When to Use
- Create Salesforce records via REST API (`SFDataFactory.createRecord()`)
- Build complex payloads with nested objects
- Merge static template data with dynamic test values

---

## When NOT to Use
- ❌ For UI-based record creation → use Page Objects + Workflows
- ❌ For simple payloads with ≤3 fields → use plain objects
- ❌ For read-only API operations → no payload needed

---

## Import
```typescript
import { PayloadBuilder } from 'playwright-custom-core';
```

---

## Core Concepts

### Template-Driven Approach
- Store base JSON templates in `test-data/api/` directory
- Templates define complete field structure with default values
- Tests override only fields requiring uniqueness or dynamic values
- All static data lives in templates, not in test code

---

## Quick Start

### Pattern 1: Load Template + Override Fields
```typescript
import { PayloadBuilder, TestDataGenerator } from 'playwright-custom-core';

// Base template: test-data/api/account-template.json
// { "Name": "Default Account", "Type": "Customer", "Industry": "Technology" }

const payload = PayloadBuilder
  .fromFile('test-data/api/account-template.json')
  .set('Name', TestDataGenerator.uniqueName('Account'))
  .set('Industry', 'Finance')
  .build();

// Result: { Name: "Account_1710345678901", Type: "Customer", Industry: "Finance" }
```

### Pattern 2: Build from Empty + Add Fields
```typescript
const payload = PayloadBuilder
  .empty<AccountPayload>()
  .set('Name', TestDataGenerator.uniqueName('Account'))
  .set('Type', 'Customer')
  .set('Industry', 'Technology')
  .build();

// Result: { Name: "Account_1710345678901", Type: "Customer", Industry: "Technology" }
```

### Pattern 3: Nested Object Updates
```typescript
// Base template has BillingAddress structure
const payload = PayloadBuilder
  .fromFile<AccountPayload>('test-data/api/account-with-address.json')
  .set('Name', TestDataGenerator.uniqueName('Account'))
  .setNested('BillingAddress.City', 'San Francisco')
  .setNested('BillingAddress.State', 'CA')
  .build();

// Result: { Name: "Account_...", BillingAddress: { ..., City: "San Francisco", State: "CA" } }
```

### Pattern 4: Merge Multiple Fields
```typescript
const dynamicFields = {
  Name: TestDataGenerator.uniqueName('Account'),
  Phone: TestDataGenerator.randomPhone(),
  Website: 'https://example.com',
};

const payload = PayloadBuilder
  .fromFile('test-data/api/account-template.json')
  .merge(dynamicFields)
  .build();
```

---

## Static Factory Methods

### `fromFile<T>(filePath: string): PayloadBuilder<T>`
Create builder from JSON template file.

**Parameters**:
- `filePath`: Absolute or relative path to JSON template

```typescript
const builder = PayloadBuilder.fromFile<ContactPayload>('test-data/api/contact-template.json');
```

---

### `fromObject<T>(base: T): PayloadBuilder<T>`
Create builder from existing object (no file).

**Parameters**:
- `base`: Base object to use as starting payload

```typescript
const basePayload = { Name: 'Test', Type: 'Customer' };
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
  .fromFile('test-data/api/account-template.json')
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

## Output Methods

### `build(): T`
Build and return the final payload object.

**Returns**: Deep-cloned final payload

**Use case**: Pass to `SFDataFactory.createRecord()` or `fetch()` body

```typescript
const payload = builder.build();
const accountId = await dataFactory.createRecord('Account', payload);
```

---

### `toJSON(pretty?: boolean): string`
Build and return payload as JSON string.

---

### `peek(): Readonly<T>`
Peek at current payload state without building (for debugging).

---

## Usage with SFDataFactory

### Complete Example — Create Account
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
  const payload = PayloadBuilder
    .fromFile<AccountPayload>('test-data/api/account-with-address.json')
    .set('Name', TestDataGenerator.uniqueName('APIAccount'))
    .set('Type', 'Customer')
    .setNested('BillingAddress.City', 'San Francisco')
    .setNested('BillingAddress.State', 'CA')
    .build();

  const accountId = await dataFactory.createRecord('Account', payload);
  // Auto-registered for cleanup

  const account = await dataFactory.fetchRecord<AccountRecord>(
    'Account', 
    accountId, 
    ['Name', 'Type', 'BillingCity', 'BillingState']
  );

  expect(account.Name).toContain('APIAccount_');
  expect(account.Type).toBe('Customer');
  expect(account.BillingCity).toBe('San Francisco');
  expect(account.BillingState).toBe('CA');
});
```

### Complete Example — Create Contact with Account
```typescript
test('should create contact linked to account via API', async () => {
  // 1. Create parent account
  const accountPayload = PayloadBuilder
    .fromFile('test-data/api/account-template.json')
    .set('Name', TestDataGenerator.uniqueName('ParentAccount'))
    .build();

  const accountId = await dataFactory.createRecord('Account', accountPayload);

  // 2. Create child contact
  const contactPayload = PayloadBuilder
    .fromFile('test-data/api/contact-template.json')
    .set('LastName', TestDataGenerator.uniqueName('Contact'))
    .set('Email', TestDataGenerator.uniqueEmail('contact'))
    .set('AccountId', accountId)  // Link to parent
    .build();

  const contactId = await dataFactory.createRecord('Contact', contactPayload);

  // 3. Verify contact is linked to account
  const contact = await dataFactory.fetchRecord('Contact', contactId, ['AccountId']);
  expect(contact.AccountId).toBe(accountId);

  // Cleanup happens automatically in teardown (child first, then parent)
});
```

---

## Advanced Patterns

### Pattern: Conditional Field Updates
```typescript
const accountType = csvRow.accountType;

const builder = PayloadBuilder.fromFile('test-data/api/account-template.json')
  .set('Name', TestDataGenerator.uniqueName('Account'));

// Conditionally add fields
if (accountType === 'Enterprise') {
  builder.set('Industry', 'Technology');
  builder.setNested('BillingAddress.Country', 'USA');
}

const payload = builder.build();
```

### Pattern: Reuse Builder with Multiple Overrides
```typescript
const baseBuilder = PayloadBuilder.fromFile('test-data/api/contact-template.json');

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

### Pattern: Combine CSV Static Data with Dynamic Overrides
```typescript
import { CsvReader } from 'playwright-custom-core';

const csvPath = path.resolve(__dirname, '../../test-data/account-api-data.csv');
const csvRow = CsvReader.readRow<Record<string, string>>(csvPath, 0)!;

const payload = PayloadBuilder
  .fromFile('test-data/api/account-template.json')
  .set('Name', TestDataGenerator.uniqueName(csvRow.namePrefix))  // Dynamic
  .set('Type', csvRow.type)                                       // Static from CSV
  .set('Industry', csvRow.industry)                               // Static from CSV
  .build();
```

---

## Best Practices

- ✅ **Store all base templates in `test-data/api/`** — centralized location
- ✅ **One template per Salesforce object** — reusable across tests
- ✅ **Use TypeScript interfaces for type safety** — catch errors at compile time
- ✅ **Override only unique/dynamic fields** — keep static data in templates
- ✅ **Use `.strict()` to enforce schema** — prevent typos and unplanned fields
- ✅ **Combine with `TestDataGenerator`** — generate unique values for required fields
- ✅ **Use `peek()` for debugging ONLY** — inspect intermediate state
- ❌ **Don't hardcode field values in test code** — use CSV or templates
- ❌ **Don't mutate original templates** — builder creates deep clones
- ❌ **Don't use for simple payloads** — plain objects sufficient for ≤3 fields

---

## Rules

- ✅ **Always load from file or object** — don't create builders manually
- ✅ **Use chaining for multiple operations** — improves readability
- ✅ **Call `build()` last** — returns final payload ready for API
- ✅ **Combine with SFDataFactory** — `dataFactory.createRecord(obj, builder.build())`
- ❌ **Don't modify returned payload** — it's a deep clone, safe to mutate if needed
- ❌ **Don't cache builders across tests** — create fresh for each test

---
