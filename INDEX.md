# Repository Index — Machine-Readable Discovery Map

> **Purpose:** Enable AI agents and humans to quickly locate the single authority for any concern.
> Scan this file first when you need to find where something is defined.
>
> This repo ships **agents**, **instructions**, and **utilities**.
> Downstream projects clone this repo and copy assets into `.github/agents/`, `.github/instructions/`, and `.github/utilities/`.

---

## Directory Structure

```
art-of-ai-instructions/
├── README.md                          # Human + AI overview, architecture, setup guide
├── INDEX.md                           # THIS FILE — discovery map
├── AGENTS.md                          # Agent registry, boundaries, dependencies
│
├── agents/                            # Centralized agent files (copied to .github/agents/)
│   ├── playwright-test-planner.agent.md
│   ├── playwright-test-generator.agent.md
│   ├── playwright-test-healer.agent.md
│   └── manual-test-step-generator.agent.md
│
├── instructions/                      # Auto-loaded instruction files (applyTo patterns)
│   ├── page-objects.instructions.md
│   ├── workflows.instructions.md
│   ├── spec-files.instructions.md
│   ├── salesforce-stability.instructions.md
│   ├── test-data.instructions.md
│   └── helper-utilities.instructions.md
│
└── utilities/                         # Helper utility reference docs (read on demand)
    ├── batch-utilities.md
    ├── csv-reader.md
    ├── email-verification-service.md
    ├── impersonation-helper.md
    ├── jira-xray-service.md
    ├── payload-builder.md
    ├── salesforce-connection.md
    ├── salesforce-login-service.md
    ├── session-manager.md
    ├── session-refresh-middleware.md
    ├── sf-data-factory.md
    └── test-data-generator.md
```

> Agents are centralized in this repo under `agents/`.
> See [AGENTS.md](AGENTS.md) for boundaries and dependencies.

---

## Single Authority Map

Each concern has exactly one authoritative file. Do not look elsewhere.

### Agents (centralized in `agents/` — copied to `.github/agents/` in consuming projects)

| Agent Name | Authority Over |
|---|---|
| `playwright-test-planner` | Test scenario design, user flow mapping, plan output |
| `playwright-test-generator` | Playwright code generation from plans (3-layer architecture) |
| `playwright-test-healer` | Debugging and fixing failing Playwright tests |
| `AI-test-case-step-generator` | Converting codegen output to manual test cases |

These agents consume centralized instructions as their authoritative source of framework rules.

### Instructions — How Code Must Be Written

> Copied to `.github/instructions/` in consuming projects.

| File | Authority Over | Auto-Loads For | Consuming Project Path |
|---|---|---|---|
| `instructions/page-objects.instructions.md` | Page Object class structure, ResilientLocator, SF-Basepage reuse, demand-driven creation, locator rules, combobox patterns, save/toast page-level mechanics | `pages/**/*.ts` | `.github/instructions/page-objects.instructions.md` |
| `instructions/workflows.instructions.md` | Workflow class structure, `this.step()` wrapping, SF-Basepage usage, demand-driven imports, one-workflow-per-scenario, record creation flow orchestration | `workflows/**/*.ts` | `.github/instructions/workflows.instructions.md` |
| `instructions/spec-files.instructions.md` | Spec file structure, zero-locator rule, CSV data loading, cleanup registration, verification chain | `tests/**/*.spec.ts` | `.github/instructions/spec-files.instructions.md` |
| `instructions/salesforce-stability.instructions.md` | Wait strategies, Lightning SPA behavior, Shadow DOM, stale elements, dynamic IDs, toast transience, common failure patterns | `tests/**/*.ts` | `.github/instructions/salesforce-stability.instructions.md` |
| `instructions/test-data.instructions.md` | CSV directory structure, file naming, static vs dynamic data, CSV isolation rule | `test-data/**` | `.github/instructions/test-data.instructions.md` |
| `instructions/helper-utilities.instructions.md` | Index of available helper utilities — read on demand, never all at once | `tests/**/*.ts` | `.github/instructions/helper-utilities.instructions.md` |

### Utilities — Reference Documentation

> Copied to `.github/utilities/` in consuming projects.
> **Usage rule:** Read only the utility file you need. Never load all utility files at once.

| File | Purpose |
|---|---|
| `utilities/batch-utilities.md` | Trigger and monitor Salesforce Apex batch jobs via Tooling API |
| `utilities/csv-reader.md` | Read static test data from CSV with type-safe mapping |
| `utilities/email-verification-service.md` | Email verification via Salesforce EmailMessage or inbox providers |
| `utilities/impersonation-helper.md` | Login as another Salesforce user via Setup → Users |
| `utilities/jira-xray-service.md` | Jira/Xray integration for test management |
| `utilities/payload-builder.md` | Type-safe API payloads from JSON templates with dynamic overrides |
| `utilities/salesforce-connection.md` | Salesforce API connection management |
| `utilities/salesforce-login-service.md` | Salesforce authentication service |
| `utilities/session-manager.md` | Browser session management |
| `utilities/session-refresh-middleware.md` | Auto-recovery from mid-test Salesforce session expiry |
| `utilities/sf-data-factory.md` | Test data CRUD, Composite API, automatic cleanup (child-before-parent) |
| `utilities/test-data-generator.md` | Unique/timestamped/random value generation for test isolation |

### Discovery Layer

| File | Purpose |
|---|---|
| `README.md` | What this repo is, architecture, how to consume it |
| `INDEX.md` | This file — machine-readable map of all files and their authority |
| `AGENTS.md` | Agent registry, boundaries, instruction dependencies |

---

## Files That Stay Local (NOT in This Repo)

| File | Location in Consuming Project | Purpose |
|---|---|---|
| `copilot-instructions.md` | `.github/copilot-instructions.md` | Project-specific context (org URL, app names, object list) |
| `local-override.md` | `.github/local-override.md` | Project-level rule overrides |
| `.env` | Project root | Credentials and environment variables |

---

## Quick Lookup Guide for AI Agents

| I need to... | Look in |
|---|---|
| Understand agent boundaries | `.github/AGENTS.md` |
| Find rules for page objects | `.github/instructions/page-objects.instructions.md` |
| Find rules for workflows | `.github/instructions/workflows.instructions.md` |
| Find rules for spec files | `.github/instructions/spec-files.instructions.md` |
| Handle Salesforce UI timing/stability | `.github/instructions/salesforce-stability.instructions.md` |
| Find rules for test data CSVs | `.github/instructions/test-data.instructions.md` |
| Find available helper utilities | `.github/instructions/helper-utilities.instructions.md` |
| Use a specific utility | `.github/utilities/<name>.md` (read only what you need) |
| Understand project-specific context | `.github/copilot-instructions.md` (local to consuming project) |
| Override centralized rules | `.github/local-override.md` (local, use sparingly) |
