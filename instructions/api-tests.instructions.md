---
applyTo: "tests/**/*.ts"
---

# API Test Rules

## When to Use

- The test DOES NOT require any UI interaction or UI validation
- The entire test can be performed COMPLETELY via API calls and response assertions

## Architecture

- Do NOT create or use any Page Objects in API tests
- Workflows MUST expose high-level business actions that the spec file calls
- Workflows internally call utility methods to perform API operations
- Workflows MUST perform all required assertions for both test logic and API responses

## Test Data Rules

- Always use `PayloadBuilder` to construct API request bodies — ensures type safety and consistency
- Use data from CSV files as base — override ONLY fields that must be unique or dynamic

## API Usage Rules

- MUST prefer `executeCompositeRequest()` when: more than one API call is needed, bulk read/write operations are involved, or references between subrequests are required for parent-child relationships
- For SINGLE API calls only, use `fetchRecord`, `deleteRecord`, `updateRecord`, `createRecord` for simplicity

## Mandatory Rules

- Always use `testStep()` for all actions in workflows — ensures proper reporting
- Prioritize efficient API usage with `executeCompositeRequest()` and minimize total API calls
- Tags follow the same format as UI tests: Jira key first, then test type — inside `test()` only
