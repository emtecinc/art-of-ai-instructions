# CsvReader Utility

## When to Use
- Load static test data (dropdown values, record types, prefixes, dates)
- Read configuration data from CSV files
- Access parameterized test data sets by index or condition
- Reuse common field values across different test scenarios

---

## When NOT to Use
- ❌ For generating unique/dynamic values → use `TestDataGenerator`
- ❌ For runtime data generation → use `TestDataGenerator`

---

## Import
```typescript
import { CsvReader } from 'playwright-custom-core';
import * as path from 'path';
```

---

## CSV File Requirements
- **Location**: `test-data/` directory at project root
- **Format**: First row must be headers (camelCase column names)
- **Naming**: `<object>-<scenario>.csv` (matches spec without action verb)
- **First column**: Always `dataSet` (row identifier: `default`, `variant1`, etc.)
- **Values**: Static values directly, dynamic field prefixes with `Prefix` suffix

---

## Methods

### `readAll<T>(filePath: string): T[]`
Read all data rows as typed objects.

```typescript
const csvPath = path.resolve(__dirname, '../../test-data/account-required-fields.csv');
const allRows = CsvReader.readAll<Record<string, string>>(csvPath);
// Returns: [{ dataSet: 'default', namePrefix: 'Acct', ... }, { dataSet: 'variant1', ... }]
```

### `readRow<T>(filePath: string, rowIndex: number): T | undefined`
Read a single row by 0-based index (excluding header).

```typescript
const csvPath = path.resolve(__dirname, '../../test-data/account-required-fields.csv');
const row = CsvReader.readRow<Record<string, string>>(csvPath, 0)!; // First data row
// Returns: { dataSet: 'default', namePrefix: 'TestAccount', type: 'Customer', ... }
```

### `findRow<T>(filePath: string, predicate: (row: T) => boolean): T | undefined`
Find first row matching a condition.

```typescript
const csvPath = path.resolve(__dirname, '../../test-data/account-required-fields.csv');
const goldRow = CsvReader.findRow<Record<string, string>>(
  csvPath,
  (r) => r.dataSet === 'gold'
);
// Returns: { dataSet: 'gold', namePrefix: 'GoldAccount', tier: 'Gold', ... }
```

---

## Usage Pattern in Test Specs

### Standard Pattern
```typescript
import * as path from 'path';
import { CsvReader, TestDataGenerator } from 'playwright-custom-core';

// 1. Resolve CSV path (remove action verb from spec name)
// Spec: create-account-required-fields.spec.ts → CSV: account-required-fields.csv
const csvPath = path.resolve(__dirname, '../../test-data/account-required-fields.csv');
const csvRow = CsvReader.readRow<Record<string, string>>(csvPath, 0)!;

// 2. Combine static CSV data with dynamic generation
const TEST_DATA = {
  name: TestDataGenerator.uniqueName(csvRow.namePrefix),  // Dynamic from prefix
  type: csvRow.type,                                       // Static from CSV
  industry: csvRow.industry,                               // Static from CSV
};
```

### Cross-Object Reuse Pattern
When different objects share common fields:

```typescript
// Reuse common fields from another CSV
const commonCsvPath = path.resolve(__dirname, '../../test-data/account-required-fields.csv');
const commonRow = CsvReader.readRow<Record<string, string>>(commonCsvPath, 0)!;

// Read scenario-specific data
const scenarioCsvPath = path.resolve(__dirname, '../../test-data/contact-all-fields.csv');
const scenarioRow = CsvReader.readRow<Record<string, string>>(scenarioCsvPath, 0)!;

const TEST_DATA = {
  accountName: TestDataGenerator.uniqueName(commonRow.namePrefix),  // Reused
  accountType: commonRow.type,                                      // Reused
  contactFirstName: scenarioRow.firstName,                          // Scenario-specific
  contactLastName: TestDataGenerator.uniqueName(scenarioRow.lastNamePrefix), // Scenario-specific
};
```

---

## CSV File Structure Example

**File**: `test-data/account-required-fields.csv`
```csv
dataSet,namePrefix,type,industry,rating
default,TestAccount,Customer,Technology,Hot
variant1,AltAccount,Partner,Finance,Warm
gold,GoldAccount,Customer,Healthcare,Hot
```

**Loading in test**:
```typescript
const csvRow = CsvReader.readRow<Record<string, string>>(csvPath, 0)!;
// csvRow = { dataSet: 'default', namePrefix: 'TestAccount', type: 'Customer', ... }

const accountName = TestDataGenerator.uniqueName(csvRow.namePrefix);
// accountName = 'TestAccount_1710345678901'
```

---

## Rules
- **Scan existing CSVs only for cross-object reuse** — when different objects share fields
- **Never hardcode data in specs** — always load from CSV