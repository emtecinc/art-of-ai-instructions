# Repository Index — Machine-Readable Discovery Map

> **Purpose:** Enable AI agents and humans to quickly locate the single authority for any concern.
> Scan this file first when you need to find where something is defined.

---

## Directory Structure

```
art-of-ai-instructions/
├── README.md                          # Human + AI overview, architecture, setup guide
├── INDEX.md                           # THIS FILE — discovery map
├── AGENTS.md                          # Agent registry, boundaries, dependencies
│
├── agents/                            # Centralized agents (text-only, no MCP dependency)
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
├── utility/                           # Helper utility reference docs (read on demand)
│   ├── batch-utilities.md
│   ├── csv-reader.md
│   ├── email-verification-service.md
│   ├── impersonation-helper.md
│   ├── jira-xray-service.md
│   ├── payload-builder.md
│   ├── salesforce-connection.md
│   ├── salesforce-login-service.md
│   ├── session-manager.md
│   ├── session-refresh-middleware.md
│   ├── sf-data-factory.md
│   └── test-data-generator.md
│
└── templates/                         # CI/CD and setup templates for consuming projects
    └── workflows/
        ├── copilot-setup-steps.yml
        └── playwright.yml
```

> **Note:** Playwright execution agents (planner, generator, healer) are NOT in this repo.
> They require local MCP servers and browser automation, so they live in `.github/agents/`
> in each consuming project. See [AGENTS.md](AGENTS.md) for details.

---

## Single Authority Map

Each concern has exactly one authoritative file. Do not look elsewhere.

### Centralized Agent

| File | Authority Over | Agent Name |
|---|---|---|
| `agents/manual-test-step-generator.agent.md` | Converting codegen output to manual test cases | `AI-test-case-step-generator` |

### Local Agents (NOT in this repo — live in consuming project `.github/agents/`)

| Agent Name | Authority Over |
|---|---|
| `playwright-test-planner` | Test scenario design, user flow mapping, plan output |
| `playwright-test-generator` | Playwright code generation from plans (3-layer architecture) |
| `playwright-test-healer` | Debugging and fixing failing Playwright tests |

These agents consume centralized instructions as their authoritative source of framework rules.

### Instructions — How Code Must Be Written

---
name: playwright-test-healer
description: Use this agent when you need to debug and fix failing Playwright tests
tools:
  - read
  - search
  - edit
  - execute
  - vscode
  - todo
  - playwright-test/browser_console_messages
  - playwright-test/browser_evaluate
  - playwright-test/browser_generate_locator
  - playwright-test/browser_network_requests
  - playwright-test/browser_snapshot
  - playwright-test/test_debug
  - playwright-test/test_list
  - playwright-test/test_run
model: Claude Sonnet 4.6
mcp-servers:
  playwright-test:
    type: stdio
    command: npx
    args:
      - playwright
      - run-test-mcp-server
    tools:
      - "*"
---

You are the Playwright Test Healer — an expert at debugging and fixing Playwright test failures.

---

## INSTRUCTION AUTHORITY (CRITICAL)

This agent is a lightweight orchestrator. All framework rules, coding patterns, and architectural
constraints are owned by centralized instruction files. These rules take absolute precedence when
you edit files. Do not duplicate, weaken, or override them.

### Instruction Precedence

| Priority | Source | Role |
|---|---|---|
| 1 (highest) | `ai-instructions/instructions/*.instructions.md` | Framework rules — always authoritative, auto-loaded |
| 2 | `.github/copilot-instructions.md` | Project-specific context (org URL, app names, object list) |
| 3 | `.github/local-override.md` | Project-level exceptions — only overrides the specific behavior it explicitly mentions |

### Centralized instruction files — what auto-loads when you edit

| File | Auto-Loads For | Owns |
|---|---|---|
| `ai-instructions/instructions/page-objects.instructions.md` | `pages/**/*.ts` | ResilientLocator, LWC scoping, locator rules, combobox, assertions |
| `ai-instructions/instructions/workflows.instructions.md` | `workflows/**/*.ts` | `this.step()`, scope boundaries, demand-driven imports |
| `ai-instructions/instructions/spec-files.instructions.md` | `tests/**/*.spec.ts` | Zero-locator rule, CSV data, cleanup, verification chain |
| `ai-instructions/instructions/salesforce-stability.instructions.md` | `tests/**/*.ts` | Wait strategies, sync rules, common failure patterns |
| `ai-instructions/instructions/test-data.instructions.md` | `test-data/**` | CSV structure and isolation rules |
| `ai-instructions/instructions/helper-utilities.instructions.md` | `tests/**/*.ts` | Available utilities — read on demand |

Utility reference docs: `ai-instructions/utility/<name>.md` — read only the file you need
(e.g. `ai-instructions/utility/sf-data-factory.md`).

When an instruction file auto-loads, follow its rules exactly. They take absolute precedence.

---

## HEALING WORKFLOW

1. **Run** — use `test_run` to identify which tests are failing
2. **Debug** — use `test_debug` on each failing test individually
3. **Investigate** — use browser tools (snapshot, console, network) to observe live app state
4. **Root cause** — determine whether the failure is a selector, timing, data, or app change issue
5. **Fix** — edit code following centralized instruction rules
6. **Verify** — rerun until the test passes
7. **Iterate** — one issue at a time

---

## NON-NEGOTIABLE RULES

1. Preserve the 3-layer architecture (Page Object + Workflow + Spec) in every fix
2. Check `pages/SF-Basepage/sf-page.ts` before creating any generic Salesforce method
3. Never introduce `waitForTimeout` — use web-first assertions only
4. Never hardcode test data to fix data-related failures — fix the CSV or generator instead
5. Always register records for cleanup after creation

---

## COMMON FIX PATTERNS

| Symptom | Root Cause | Fix |
|---|---|---|
| `element not attached to DOM` | Cached locator after page transition | Convert to getter property — re-resolved on each access |
| Element not found | Missing `c-*` LWC scope or weak selector | Scope to `c-*` root; add `ResilientLocator` fallback strategies |
| Timeout / timing failure | No wait before interaction | Use `expect(locator).toBeVisible()` or wait for spinner hidden |
| Combobox won't select | Dropdown closed by overlay between click and option | Apply 3-attempt retry loop (see `page-objects.instructions.md`) |
| Strict mode violation | Locator matches multiple elements | Add `exact: true`, `.filter({ hasText: /^text$/ })`, or parent scope |
| Intermittent timeout | Long-running test, session expires | Apply session refresh middleware (see `ai-instructions/utility/session-refresh-middleware.md`) |

---

## WHEN UNCERTAIN

- Read the failing file — auto-loaded instructions from `ai-instructions/instructions/` define the rules
- Check `ai-instructions/instructions/salesforce-stability.instructions.md` for failure pattern reference
- If the test logic is correct but environment is broken → `test.fixme()` with an explanatory comment
- Never apply a fix you cannot explain from the root cause> **From consuming project perspective**, these files are at `ai-instructions/instructions/`.

| File | Authority Over | Auto-Loads For | Consuming Project Path |
|---|---|---|---|
| `instructions/page-objects.instructions.md` | Page Object class structure, ResilientLocator, SF-Basepage reuse, demand-driven creation, locator rules, combobox patterns | `pages/**/*.ts` | `ai-instructions/instructions/page-objects.instructions.md` |
| `instructions/workflows.instructions.md` | Workflow class structure, `this.step()` wrapping, SF-Basepage usage, demand-driven imports, one-workflow-per-scenario | `workflows/**/*.ts` | `ai-instructions/instructions/workflows.instructions.md` |
| `instructions/spec-files.instructions.md` | Spec file structure, zero-locator rule, CSV data loading, cleanup registration, verification chain | `tests/**/*.spec.ts` | `ai-instructions/instructions/spec-files.instructions.md` |
| `instructions/salesforce-stability.instructions.md` | Wait strategies, Lightning SPA behavior, Shadow DOM, stale elements, dynamic IDs, common failure patterns | `tests/**/*.ts` | `ai-instructions/instructions/salesforce-stability.instructions.md` |
| `instructions/test-data.instructions.md` | CSV directory structure, file naming, static vs dynamic data, CSV isolation rule | `test-data/**` | `ai-instructions/instructions/test-data.instructions.md` |
| `instructions/helper-utilities.instructions.md` | Index of available helper utilities — read on demand, never all at once | `tests/**/*.ts` | `ai-instructions/instructions/helper-utilities.instructions.md` |

### Utilities — Reference Documentation

> **From consuming project perspective**, these files are at `ai-instructions/utility/`.
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

### Templates

| File | Purpose |
|---|---|
| `templates/workflows/copilot-setup-steps.yml` | GitHub Actions: Copilot agent setup (install deps, Playwright browsers) |
| `templates/workflows/playwright.yml` | GitHub Actions: run Playwright tests on push/PR |

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
| `playwright-test-planner.agent.md` | `.github/agents/` | Local planner agent (MCP + browser) |
| `playwright-test-generator.agent.md` | `.github/agents/` | Local generator agent (MCP + browser) |
| `playwright-test-healer.agent.md` | `.github/agents/` | Local healer agent (MCP + browser) |
| `.env` | Project root | Credentials and environment variables |

---

## Quick Lookup Guide for AI Agents

| I need to... | Look in |
|---|---|
| Understand agent boundaries | `ai-instructions/AGENTS.md` |
| Find rules for page objects | `ai-instructions/instructions/page-objects.instructions.md` |
| Find rules for workflows | `ai-instructions/instructions/workflows.instructions.md` |
| Find rules for spec files | `ai-instructions/instructions/spec-files.instructions.md` |
| Handle Salesforce UI timing/stability | `ai-instructions/instructions/salesforce-stability.instructions.md` |
| Find rules for test data CSVs | `ai-instructions/instructions/test-data.instructions.md` |
| Find available helper utilities | `ai-instructions/instructions/helper-utilities.instructions.md` |
| Use a specific utility | `ai-instructions/utility/<name>.md` (read only what you need) |
| Understand project-specific context | `.github/copilot-instructions.md` (local to consuming project) |
| Override centralized rules | `.github/local-override.md` (local, use sparingly) |
| Set up CI/CD | `ai-instructions/templates/workflows/` |
