---
applyTo: "tests/**/*.ts, workflows/**/*.ts"
---

# UTILITIES TO HELP WITH PLAYWRIGHT TEST PLANNING AND GENERATION – STRICT ACCESS RULES
- ALWAYS read only the specific files listed below that are required for the current task.
- Read ALL the contents of the file and carefuly analyze relevant content for the current task.
- ALWAYS USE only the relevant contents from the file as needed; do NOT treat all file contents as context.
---

| # | File | Purpose |
|---|------|-------------|
| 1 | `.github/utilities/batch-utilities.md` | Trigger and monitor Salesforce Apex batch jobs via Tooling API with polling-based completion tracking. |
| 2 | `.github/utilities/csv-reader.md` | Reads static test data from CSV files with type-safe object mapping and use it in tests. |
| 3 | `.github/utilities/email-verification-service.md` | Email verification service that orchestrates email validation by querying Salesforce EmailMessage object and/or confirming delivery to inbox (Mailosaur, Outlook, etc.). |
| 4 | `.github/utilities/impersonation-helper.md` | Static utility for logging in as another Salesforce user via Setup → Users → Login, and logging back as admin if required. Operates on the same browser context without creating new sessions. |
| 5 | `.github/utilities/session-refresh-middleware.md` | Detects Salesforce server-side session expiry mid-test and auto-recovers. Useful for long-running tests, with idle time > 15 minutes. |
| 6 | `.github/utilities/sf-data-factory.md` | Comprehensive Salesforce test data management — CRUD operations, Composite API, and automatic cleanup of records. Supports both UI-created and API-created records with reverse-order deletion (child-before-parent). |
| 7 | `.github/utilities/test-data-generator.md` | Generates unique, timestamped, or random values at runtime to prevent duplicate-detection collisions in Salesforce tests. |
| 8 | `.github/utilities/payload-builder.md` | Build type-safe API payloads with dynamic field overrides, to use with API calls in `sf-data-factory`. |

## UTILITY USAGE - CRITICAL GUIDELINES
- [ ] These utilities are intended ONLY for use during the test files generation phase
- [ ] - Under no circumstances may these utilities be used as a pretext, workaround, or justification to bypass, skip, or shorten any steps in the browser interaction and code capture process that must occur BEFORE test file generation.
- [ ] The methods form these utilities are to called ONLY in `workflows`. They are NOT to be called in page objects or spec files.