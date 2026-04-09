# TestDataGenerator Utility

## When to Use
- Generate unique record names, serial numbers, emails
- Create runtime-unique values from CSV prefixes
- Prevent "Duplicate Value" errors in Salesforce
- Generate random phone numbers or integers for test data

---

## When NOT to Use
- ❌ For static dropdown/picklist values → use CSV
- ❌ For fixed configuration data → use CSV
- ❌ For record type names → use CSV
- ❌ For hardcoding any data in spec files → use CSV + TestDataGenerator pattern

---

## Import
```typescript
import { TestDataGenerator } from 'playwright-custom-core';
```

---

## Methods

### `uniqueName(prefix: string): string`
Generate unique string with prefix and timestamp.

```typescript
const accountName = TestDataGenerator.uniqueName('TestAccount');
// Returns: 'TestAccount_1710345678901'
```

**Usage with CSV prefix**:
```typescript
const csvRow = CsvReader.readRow(csvPath, 0)!;
const accountName = TestDataGenerator.uniqueName(csvRow.namePrefix);
// CSV namePrefix: 'Acct' → Returns: 'Acct_1710345678901'
```

### `uniqueSerial(prefix?: string): string`
Generate unique serial number with prefix.

```typescript
const serialNumber = TestDataGenerator.uniqueSerial('SN');
// Returns: 'SN-1710345678901'

const defaultSerial = TestDataGenerator.uniqueSerial();
// Returns: 'SN-1710345678901' (default prefix: 'SN')
```

### `uniqueEmail(prefix?: string, domain?: string): string`
Generate unique email address.

```typescript
const email = TestDataGenerator.uniqueEmail('testuser');
// Returns: 'testuser_1710345678901@test.example.com'

const customEmail = TestDataGenerator.uniqueEmail('admin', 'acme.com');
// Returns: 'admin_1710345678901@acme.com'
```

### `randomInt(min: number, max: number): number`
Generate random integer between min and max (inclusive).

```typescript
const age = TestDataGenerator.randomInt(18, 65);
// Returns: random number between 18 and 65

const priority = TestDataGenerator.randomInt(1, 10);
// Returns: random number between 1 and 10
```

### `randomPhone(): string`
Generate random 10-digit phone number.

```typescript
const phone = TestDataGenerator.randomPhone();
// Returns: '5551234567' (format: 555XXXXXXX)
```

### `resolvePlaceholders<T>(data: T): T`
Batch-resolve placeholder tokens in an object.

**Supported placeholders**:
- `{{UNIQUE}}` → timestamp
- `{{UNIQUE_NAME:prefix}}` → `prefix_timestamp`
- `{{UNIQUE_SERIAL:prefix}}` → `prefix-timestamp`
- `{{UNIQUE_EMAIL}}` → unique email
- `{{RANDOM_PHONE}}` → random phone

```typescript
const data = {
  name: '{{UNIQUE_NAME:Account}}',
  serial: '{{UNIQUE_SERIAL:SN}}',
  email: '{{UNIQUE_EMAIL}}',
  phone: '{{RANDOM_PHONE}}',
  timestamp: '{{UNIQUE}}',
};

const resolved = TestDataGenerator.resolvePlaceholders(data);
// Returns:
// {
//   name: 'Account_1710345678901',
//   serial: 'SN-1710345678901',
//   email: 'testuser_1710345678901@test.example.com',
//   phone: '5559876543',
//   timestamp: '1710345678901'
// }
```

---

## When to Use Each Method

| Data Type | Method | Example |
|-----------|--------|---------|
| Record names | `uniqueName()` | Account Name, Opportunity Name |
| Serial/reference numbers | `uniqueSerial()` | SLA Serial, Ticket Number |
| Email addresses | `uniqueEmail()` | Contact Email, Lead Email |
| Phone numbers | `randomPhone()` | Contact Phone, Account Phone |
| Age, quantity, priority | `randomInt()` | Age: 18-65, Quantity: 1-100 |
| Batch processing | `resolvePlaceholders()` | Multiple fields with placeholders |

---

## CSV Prefix Convention

Store prefixes in CSV with `Prefix` suffix:

**CSV Structure**:
```csv
dataSet,namePrefix,emailPrefix,serialPrefix,type,industry
default,TestAccount,testuser,SN,Customer,Technology
variant1,AltAccount,altuser,ALT,Partner,Finance
```

**Usage**:
```typescript
const csvRow = CsvReader.readRow(csvPath, 0)!;

const TEST_DATA = {
  name: TestDataGenerator.uniqueName(csvRow.namePrefix),           // 'TestAccount_...'
  email: TestDataGenerator.uniqueEmail(csvRow.emailPrefix),        // 'testuser_...@test.example.com'
  serial: TestDataGenerator.uniqueSerial(csvRow.serialPrefix),     // 'SN-...'
  type: csvRow.type,                                                // 'Customer' (static)
  industry: csvRow.industry,                                        // 'Technology' (static)
};
```

---

## Rules
- **Only unique fields need TestDataGenerator** — all other fields stay static from CSV
- **Combine with CSV prefixes** — never hardcode prefixes in spec files
- **Use appropriate method** — `uniqueName()` for names, `uniqueSerial()` for serials, etc.
- **Placeholder pattern for batch** — use `resolvePlaceholders()` when multiple fields need dynamics
