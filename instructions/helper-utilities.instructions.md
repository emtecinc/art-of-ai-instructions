---
applyTo: "tests/**/*.ts"
---

# Utilities to help with playwright test planning and generation
- ALWAYS read required files only.
- Read and use the contents of the files listed below on demand, NEVER read all the files.
- Use only the relevant sections from the files as needed; do not treat all file contents as context.

---

| # | File | Purpose |
|---|------|-------------|
| 1 | `utilities/batch-utilities.md` | Trigger and monitor Salesforce Apex batch jobs via Tooling API with polling-based completion tracking.|
| 2 | `utilities/csv-reader.md` | Reads static test data from CSV files with type-safe object mapping and use it in tests |
| 3 | `utilities/email-verification-service.md` | Email verification service that orchestrates email validation by querying Salesforce EmailMessage object and/or confirming delivery to inbox (Mailosaur, Outlook, etc.). |
| 4 | `utilities/impersonation-helper.md` | Static utility for logging in as another Salesforce user via Setup → Users → Login, and logging back as admin if required. Operates on the same browser context without creating new sessions. |
| 5 | `utilities/session-refresh-middleware.md` | Detects Salesforce server-side session expiry mid-test and auto-recovers. Useful for long-running tests, with idle time > 15 minutes. |
| 6 | `utilities/sf-data-factory.md` | Comprehensive Salesforce test data management — CRUD operations, Composite API, and automatic cleanup of records. Supports both UI-created and API-created records with reverse-order deletion (child-before-parent). |
| 7 | `utilities/test-data-generator.md` | Generates unique, timestamped, or random values at runtime to prevent duplicate-detection collisions in Salesforce tests. |
| 8 | `utilities/payload-builder.md` | Build type-safe API payloads from JSON templates with dynamic field overrides. Designed for Salesforce REST API record creation where a base JSON template defines the full structure and tests only override fields requiring unique/dynamic values. |