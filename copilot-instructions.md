# Salesforce Automation Framework — Project Context & Instruction Bridge
This file provides project-specific context. Framework rules live in `.github/instructions/` — this file never replaces or duplicates them.
## Architecture
| Layer | Location | Role |
|-|-|-|
| **Framework Rules** | `.github/instructions/*.instructions.md` | Single authority — auto-loaded via `applyTo` |
| **Utility Helpers** | `.github/utilities/*.md` | Reference-only — read on demand |
| **Project Context** | `.github/copilot-instructions.md` (this file) | Org details, objects, conventions |
| **Agent Definitions** | `.github/agents/*.agent.md` | Tool/MCP config + execution workflow |
| **Project Overrides** | `.github/local-override.md` | Specific exceptions (use sparingly) |
### Instruction Precedence
`.github/instructions/*.instructions.md` > `.github/copilot-instructions.md` > `.github/local-override.md`
## Where to Look
| Concern | File |
|-|-|
| Page objects, locators, assertions, `captureScreenshot()` | `.github/instructions/page-objects.instructions.md` |
| Workflow rules, `this.testStep()`, boundaries | `.github/instructions/workflows.instructions.md` |
| Spec rules, `try/catch/finally`, `addLocatorHandler`, cleanup | `.github/instructions/spec-files.instructions.md` |
| Salesforce timing, stability, failure patterns | `.github/instructions/salesforce-stability.instructions.md` |
| CSV structure and test data isolation | `.github/instructions/test-data.instructions.md` |
| Utility Helpers | `.github/instructions/helper-utilities.instructions.md` |
| Pure API, no UI interaction | `.github/instructions/api-tests.instructions.md` |
## Forbidden Behaviors
- **No hallucination** — Never invent selectors, methods, or flows not verified in browser or codebase
- **No hardcoded data** — All test data via CSV + `TestDataGenerator`
- **No** `waitForTimeout()`, `pressSequentially`, ID selectors, class selectors, XPath
- **No** `expect()` in specs or workflows — page objects only
- **No** toast verification inside create workflow methods — separate `verifySuccessToast()` method
- **No** cleanup outside `finally` block
- **No** cached locators across page transitions — use getter properties
- **No** `resilientLocator.click()` — always call `.getLocator()` first
- **No** partial or incomplete file reading — ALWAYS process the entire file until EOF is reached and then retain all required information for proper context.
## Utility Usage
Read from `.github/utilities/` only when the scenario requires it. Index at `.github/instructions/helper-utilities.instructions.md`.
## Salesforce Org
| Property | Value |
|-|-|
| Base URL | `https://<your-instance>.lightning.force.com` |
| API version | `v65.0` |
| Env variable | `BASE_URL` — never hardcode |
## Application Context
| Property | Value |
|-|-|
| App name | `<Your App Name>` |
| Navigation | App Launcher → `<App Name>` |
| Custom namespace | `c-` (LWC) / `<ns>__` (managed package) |
## File Generation Order
| Step | File | Rule |
|-|-|-|
| 1 | CSV | Define inputs first |
| 2 | Page Object(s) | Locators + actions + assertions |
| 3 | Workflow | Business logic — calls page methods |
| 4 | Spec | Orchestrates workflow + cleanup |
**Always check if file exists before creating — extend, never rewrite.**
## Custom Behaviors and Exceptions
### Custom LWC Wrappers
| Feature | LWC Root | Notes |
|-|-|-|
| <!-- Case Form --> | `c-case-create-form` | Scopes all locators |
## CI/CD
| Property | Value |
|-|-|
| CI file | `.github/workflows/playwright.yml` |
| Test command | `npx playwright test` |
| Environments | `DEV`, `QA`, `UAT` via `BASE_URL` |