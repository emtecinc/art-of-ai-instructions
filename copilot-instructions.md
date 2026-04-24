# Salesforce Automation Framework — Project Context

This file contains project-specific constants. **Framework rules and guidelines live in `.github/rules/`.** Do NOT read rule files upfront — retrieve them only during the appropriate agent pipeline stage.

---

## 🏛️ Rule Directory (Where to find framework logic)

| Rule File | Purpose |
|---|---|
| `.github/rules/01-framework-architecture.md` | 3-layer architecture rules, Spec/Workflow/Page Object/CSV templates. |
| `.github/rules/02-salesforce-ui-rules.md` | DOM interaction, ResilientLocator, wait priorities, toast & combobox handling. |
| `.github/rules/03-utilities-dictionary.md` | Signatures and use-cases for custom utilities (DataFactory, Impersonation, etc.). |

---

## ☁️ Salesforce Org Constants

| Property | Value |
|---|---|
| Base URL | `https://<your-instance>.lightning.force.com` |
| API version | `v60.0` |
| Env variable | `BASE_URL` — never hardcode in specs |
| App name | `<Your App Name>` |
| Navigation | App Launcher → `<App Name>` |
| Custom namespace | `c-` (LWC) / `<ns>__` (managed package) |

---

## 📦 Objects in Scope

| Object Label | API Name | Type | App Launcher Search |
|---|---|---|---|
| Account | `Account` | Standard | `Accounts` |
| Contact | `Contact` | Standard | `Contacts` |
| Opportunity | `Opportunity` | Standard | `Opportunities` |
| Case | `Case` | Standard | `Cases` |

---

## 🏷️ Naming Conventions

| Item | Pattern | Example |
|------|---------|---------|
| Page file | `<object>-<purpose>-page.ts` | `opportunity-list-page.ts` |
| Workflow file | `<action>-<object>-<scenario>-workflow.ts` | `create-opportunity-required-fields-workflow.ts` |
| Spec file | `<action/scenario>.spec.ts` | `create-opportunity-required-fields.spec.ts` |
| CSV file | `<object>-<scenario>.csv` | `account-required-fields.csv` |
| Test title | `should` + business description | `should create opportunity with required fields @smoke` |
| Directories | **Singular** object names | `pages/account/`, `test-data/account/` |
| Test data prefix | `Test_` | `Test_<timestamp>` |
| Email domain | `@test.example.com` | |
| Date format (CSV) | `MM/DD/YYYY` | |

---

## 🧩 Custom Behaviors and Exceptions

### Known UI Behaviors
- - ### Custom LWC Wrappers
*(Note: Elements inside these must be scoped using `.locator('c-wrapper-name')` in `ResilientLocator`)*

| Feature | LWC Root | Notes |
|---|---|---|
| | `c-case-create-form` | Scopes all locators |

---

## 🚀 CI/CD

| Property | Value |
|---|---|
| CI file | `.github/workflows/playwright.yml` |
| Test command | `npx playwright test` |
| Environments | `DEV`, `QA`, `UAT` via `BASE_URL` |