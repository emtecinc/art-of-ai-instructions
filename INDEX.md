# Repository Index — Machine-Readable Discovery Map

> **Purpose:** Enable AI agents and humans to quickly locate the single authority for any concern.
> Scan this file first when you need to find where something is defined.
>
> **This repo is instructions-only.** It does NOT contain or ship any agent files.
> Centralized instruction root: `.github/ai-instructions/`

---

## Directory Structure

```
art-of-ai-instructions/
├── README.md                          # Human + AI overview, architecture, setup guide
├── INDEX.md                           # THIS FILE — discovery map
├── AGENTS.md                          # Agent registry, boundaries, dependencies (reference only)
│
├── instructions/                      # Auto-loaded instruction files (applyTo patterns)
│   ├── page-objects.instructions.md
│   ├── workflows.instructions.md
│   ├── spec-files.instructions.md
│   ├── salesforce-stability.instructions.md
│   ├── test-data.instructions.md
│   └── helper-utilities.instructions.md
│
└── utility/                           # Helper utility reference docs (read on demand)
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

> **Note:** ALL agents (planner, generator, healer, manual test step generator) are LOCAL.
> They live in `.github/agents/` in each consuming project.
> See [AGENTS.md](AGENTS.md) for boundaries and dependencies.

---

## Single Authority Map

Each concern has exactly one authoritative file. Do not look elsewhere.

### Agents (NOT in this repo — all live in consuming project `.github/agents/`)

| Agent Name | Authority Over |
|---|---|
| `playwright-test-planner` | Test scenario design, user flow mapping, plan output |
| `playwright-test-generator` | Playwright code generation from plans (3-layer architecture) |
| `playwright-test-healer` | Debugging and fixing failing Playwright tests |
| `AI-test-case-step-generator` | Converting codegen output to manual test cases |

These agents consume centralized instructions as their authoritative source of framework rules.

### Instructions — How Code Must Be Written

> When consumed under `.github/ai-instructions/`, these files are at `.github/ai-instructions/instructions/`.

| File | Authority Over | Auto-Loads For | Consuming Project Path |
|---|---|---|---|
| `instructions/page-objects.instructions.md` | Page Object class structure, ResilientLocator, SF-Basepage reuse, demand-driven creation, locator rules, combobox patterns, save/toast page-level mechanics | `pages/**/*.ts` | `.github/ai-instructions/instructions/page-objects.instructions.md` |
| `instructions/workflows.instructions.md` | Workflow class structure, `this.step()` wrapping, SF-Basepage usage, demand-driven imports, one-workflow-per-scenario, record creation flow orchestration | `workflows/**/*.ts` | `.github/ai-instructions/instructions/workflows.instructions.md` |
| `instructions/spec-files.instructions.md` | Spec file structure, zero-locator rule, CSV data loading, cleanup registration, verification chain | `tests/**/*.spec.ts` | `.github/ai-instructions/instructions/spec-files.instructions.md` |
| `instructions/salesforce-stability.instructions.md` | Wait strategies, Lightning SPA behavior, Shadow DOM, stale elements, dynamic IDs, toast transience, common failure patterns | `tests/**/*.ts` | `.github/ai-instructions/instructions/salesforce-stability.instructions.md` |
| `instructions/test-data.instructions.md` | CSV directory structure, file naming, static vs dynamic data, CSV isolation rule | `test-data/**` | `.github/ai-instructions/instructions/test-data.instructions.md` |
| `instructions/helper-utilities.instructions.md` | Index of available helper utilities — read on demand, never all at once | `tests/**/*.ts` | `.github/ai-instructions/instructions/helper-utilities.instructions.md` |

### Utilities — Reference Documentation

> When consumed under `.github/ai-instructions/`, these files are at `.github/ai-instructions/utility/`.
> **Usage rule:** Read only the utility file you need. Never load all utility files at once.

| File | Purpose |
|---|---|
| `utility/batch-utilities.md` | Trigger and monitor Salesforce Apex batch jobs via Tooling API |
| `utility/csv-reader.md` | Read static test data from CSV with type-safe mapping |
| `utility/email-verification-service.md` | Email verification via Salesforce EmailMessage or inbox providers |
| `utility/impersonation-helper.md` | Login as another Salesforce user via Setup → Users |
| `utility/jira-xray-service.md` | Jira/Xray integration for test management |
| `utility/payload-builder.md` | Type-safe API payloads from JSON templates with dynamic overrides |
| `utility/salesforce-connection.md` | Salesforce API connection management |
| `utility/salesforce-login-service.md` | Salesforce authentication service |
| `utility/session-manager.md` | Browser session management |
| `utility/session-refresh-middleware.md` | Auto-recovery from mid-test Salesforce session expiry |
| `utility/sf-data-factory.md` | Test data CRUD, Composite API, automatic cleanup (child-before-parent) |
| `utility/test-data-generator.md` | Unique/timestamped/random value generation for test isolation |

### Discovery Layer

| File | Purpose |
|---|---|
| `README.md` | What this repo is, architecture, how to consume it |
| `INDEX.md` | This file — machine-readable map of all files and their authority |
| `AGENTS.md` | Agent registry, boundaries, instruction dependencies (reference only — no agent files here) |

---

## Files That Stay Local (NOT in This Repo)

| File | Location in Consuming Project | Purpose |
|---|---|---|
| `copilot-instructions.md` | `.github/copilot-instructions.md` | Project-specific context (org URL, app names, object list) |
| `local-override.md` | `.github/local-override.md` | Project-level rule overrides |
| `playwright-test-planner.agent.md` | `.github/agents/` | Local planner agent (MCP + browser) |
| `playwright-test-generator.agent.md` | `.github/agents/` | Local generator agent (MCP + browser) |
| `playwright-test-healer.agent.md` | `.github/agents/` | Local healer agent (MCP + browser) |
| `manual-test-step-generator.agent.md` | `.github/agents/` | Local manual test case agent |
| `.env` | Project root | Credentials and environment variables |

---

## Quick Lookup Guide for AI Agents

| I need to... | Look in |
|---|---|
| Understand agent boundaries | `.github/ai-instructions/AGENTS.md` |
| Find rules for page objects | `.github/ai-instructions/instructions/page-objects.instructions.md` |
| Find rules for workflows | `.github/ai-instructions/instructions/workflows.instructions.md` |
| Find rules for spec files | `.github/ai-instructions/instructions/spec-files.instructions.md` |
| Handle Salesforce UI timing/stability | `.github/ai-instructions/instructions/salesforce-stability.instructions.md` |
| Find rules for test data CSVs | `.github/ai-instructions/instructions/test-data.instructions.md` |
| Find available helper utilities | `.github/ai-instructions/instructions/helper-utilities.instructions.md` |
| Use a specific utility | `.github/ai-instructions/utility/<name>.md` (read only what you need) |
| Understand project-specific context | `.github/copilot-instructions.md` (local to consuming project) |
| Override centralized rules | `.github/local-override.md` (local, use sparingly) |
