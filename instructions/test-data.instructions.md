---
applyTo: "test-data/**"
---
# Test Data Rules
## Directory Structure (CRITICAL)
Test data is organized by object in subdirectories using **singular** object names:
```
test-data/
├── account/
│   ├── account-required-fields.csv
│   └── account-billing-address.csv
├── opportunity/
│   ├── opportunity-required-fields.csv
│   └── opportunity-details-tab-display.csv
└── contact/
    └── contact-required-fields.csv
```
**Always singular:** `test-data/account/`, never `test-data/accounts/`

## CSV File Naming
**One CSV file per test spec.** Derive name from spec filename by removing the action verb prefix:
| Spec file | CSV file | Prefix removed |
|-|-|-|
| `create-account-required-fields.spec.ts` | `account-required-fields.csv` | `create-` |
| `edit-account-workflow.spec.ts` | `account-workflow.csv` | `edit-` |
| `validation-missing-account-name.spec.ts` | `account-missing-account-name.csv` | `validation-` |
| `cancel-account-creation.spec.ts` | `account-creation.csv` | `cancel-` |
| `details-tab-display.spec.ts` | `account-details-tab-display.csv` | _(none — add object prefix)_ |

Common action prefixes to remove: `create-`, `update-`, `delete-`, `submit-`, `edit-`, `verify-`, `validation-`, `cancel-`

**If the spec name has no action prefix**, prepend the object name: `details-tab-display.spec.ts` → `account-details-tab-display.csv`

## CSV Path in Specs
```typescript
// Object directory is SINGULAR — matches test-data/<object>/
const csvPath = path.resolve(__dirname, '../../test-data/account/account-required-fields.csv');
```

## CSV Format Rules
- **Row 1:** Header row with camelCase column names
- **Column 1:** `dataSet` identifier (optional) — `default`, `variant1`, etc.
- **Static values:** Stored directly (`Yes`, `Gold`, `Prospecting`)
- **Dynamic prefixes:** Column names end with `Prefix` for fields requiring uniqueness
```csv
dataSet,namePrefix,type,status,serialPrefix
default,TestAccount,Customer,Active,SN
```

## Static vs Dynamic Data
| Data Type | Source | Example |
|-|-|-|
| Dropdown/picklist values | CSV (static) | `Prospecting`, `Active` |
| Fixed dates, numbers | CSV (static) | `06/30/2026`, `28` |
| Field value prefixes | CSV (static prefix) | `TestPrefix` |
| Full record names | `TestDataGenerator.uniqueName(prefix)` | `TestPrefix_1710345678901` |
| Serial numbers | `TestDataGenerator.uniqueSerial(prefix)` | `SN-1710345678901` |
| Email addresses | `TestDataGenerator.uniqueEmail(prefix)` | `user_171034@test.example.com` |
| Phone numbers | `TestDataGenerator.randomPhone()` | `5551234567` |

## CSV Isolation Rule
**Every spec file gets its own dedicated CSV. Never share CSVs between specs.**
- New spec → new CSV (even if identical data exists elsewhere)
- Duplicate values into new CSVs — no cross-file reads
- This ensures full test isolation and parallel execution safety

## Test Data MANDATORY Rules
- **Never hardcode test data in spec files** — all data via CSV + `TestDataGenerator`
- **Sensitive data** (passwords, URLs) goes in `process.env`, never in CSV