---
applyTo: "tests/**/*.ts"
---
# When to use
- The test DOES NOT require any UI interaction or UI validation.
- The entire test can be PERFORMED COMPLETELY via API calls and response assertions.

# API Test Best Practices and Guidelines
## Page Object Model
- Do NOT create or use any Page Objects in API tests.
- Call high-level methods directly from the `.spec` files.
- Workflows MUST expose the high-level business actions that the spec file calls, and internally call utility methods to perform API operations.
- Workflows MUST perform all required assertions for both the test logic and API responses..

## Test Data Rules
- Always use `PayloadBuilder` to construct the required API request bodies to ENSURE type safety and consistency.
- Use data from CSV files as base. Override ONLY fields that must be unique or dynamic for the specific test case.

## API usage rules
- MUST prefer `executeCompositeRequest()` in cases where → More than one API call is needed at a time, Bulk read or write operations are involved, References between subrequests are required for parent-child relationships.
- For SINGLE API call only, use `fetchRecord`, `deleteRecord`, `updateRecord` and`createRecord` for simplicity.

## Critical Rules
- Always use `testStep()` for all actions in workflows, to ensure proper reporting.
- Prioritise efficient API usage with `executeCompositeRequest()` and actively minimize number of API calls made.