---
applyTo: "tests/**/*.ts, workflows/**/*.ts"
---

# Utilities for Playwright Test Planning and Generation

- Read ONLY the specific utility files required for the current task.
- Read the full contents of the file and carefully analyze relevant sections.
- Use only the relevant content as needed; do NOT treat all file contents as context.

---

| # | File | Purpose |
|---|------|-------------|
| 1 | `utilities/batch-utilities.md` | Trigger and monitor Salesforce Apex batch jobs via Tooling API with polling-based completion tracking. |
| 2 | `utilities/csv-reader.md` | Reads static test data from CSV files with type-safe object mapping and use it in tests. |
| 3 | `utilities/email-verification-service.md` | Email verification service that orchestrates email validation by querying Salesforce EmailMessage object and/or confirming delivery to inbox (Mailosaur, Outlook, etc.). |
| 4 | `utilities/impersonation-helper.md` | Static utility for logging in as another Salesforce user via Setup → Users → Login, and logging back as admin if required. Operates on the same browser context without creating new sessions. |
| 5 | `utilities/session-refresh-middleware.md` | Detects Salesforce server-side session expiry mid-test and auto-recovers. Useful for long-running tests, with idle time > 15 minutes. |
| 6 | `utilities/sf-data-factory.md` | Comprehensive Salesforce test data management — CRUD operations, Composite API, and automatic cleanup of records. Supports both UI-created and API-created records with reverse-order deletion (child-before-parent). |
| 7 | `utilities/test-data-generator.md` | Generates unique, timestamped, or random values at runtime to prevent duplicate-detection collisions in Salesforce tests. |
| 8 | `utilities/payload-builder.md` | Build type-safe API payloads with dynamic field overrides, to use with API calls in `sf-data-factory`. |

## Utility Calling Rules

- Utility methods are called in `workflow` and `spec` files as required by the task.
- Do NOT call utility methods in page objects.