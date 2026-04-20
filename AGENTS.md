# Agent Registry

Reference documentation for all agents in the system — their purpose, boundaries, and instruction dependencies.

> This repo ships **agents**, **instructions**, and **utilities**.
> Downstream projects clone this repo and copy assets into `.github/agents/`, `.github/instructions/`, and `.github/utilities/`.
> This file serves as a reference registry so agents and humans can understand boundaries and dependencies.

---

## Architecture

All agents are **centralized** in this repo under `agents/` and are copied to `.github/agents/` in each consuming project.

| Agent | Centralized Location | Consuming Project Location |
|---|---|---|
| `playwright-test-planner` | `agents/playwright-test-planner.agent.md` | `.github/agents/` |
| `playwright-test-generator` | `agents/playwright-test-generator.agent.md` | `.github/agents/` |
| `playwright-test-healer` | `agents/playwright-test-healer.agent.md` | `.github/agents/` |
| `AI-test-case-step-generator` | `agents/manual-test-step-generator.agent.md` | `.github/agents/` |

### Pipeline

```
Planner ──(plan)──► Generator ──(code)──► Healer
```

The **Manual Test Step Generator** (`AI-test-case-step-generator`) is standalone — it does not participate in the Playwright pipeline.

### How Agents Consume Centralized Instructions

Agents are **lightweight orchestrators**. They define:
- Tool configuration (MCP server, browser tools)
- Workflow sequence (what to do, in what order)
- Execution-specific behavior (browser-first verification, save locations)

**All framework rules, coding patterns, and architectural constraints live in centralized instruction files** (consumed under `.github/instructions/`). Agents MUST:

1. Follow auto-loaded instruction rules without overriding them
2. Use `.github/copilot-instructions.md` for project-specific context only
3. Use `.github/local-override.md` for project-specific exceptions only
4. Never duplicate centralized instruction content

---

## Agent Reference — Boundaries and Dependencies

All agents below are centralized in this repo under `agents/`. They are copied to `.github/agents/` in each consuming project.
This section documents their boundaries and instruction dependencies for reference.

---

### playwright-test-planner

**Purpose:** Explore a live Salesforce application and produce comprehensive, business-readable test plans that the Generator can implement directly.

**MCP Server:** `playwright-test` (browser tools + `planner_setup_page` + `planner_save_plan`)

#### Scope Boundaries

| ✅ Responsible For | ❌ Must NOT Include |
|---|---|
| Test scenario design | Locator strategies or CSS selectors |
| User flow mapping | ResilientLocator, BasePage, BaseWorkflow references |
| Edge case identification | CSV structure or test data design |
| Suite and scenario organization | Cleanup/teardown logic or TypeScript code |
| Identifying relevant utilities | Which page files to create (Generator decides on demand) |

#### Instruction Dependencies

| Dependency | Consuming Project Path | Why |
|---|---|---|
| `instructions/helper-utilities.instructions.md` | `.github/instructions/helper-utilities.instructions.md` | Identify available utilities for test implementation |
| `copilot-instructions.md` (local) | `.github/copilot-instructions.md` | Project-specific app names, object list, org context |

#### Output

Test plan saved to `specs/` — contains scenario titles, step-by-step user actions, expected outcomes, tags.

---

### playwright-test-generator

**Purpose:** Implement test plans as production-ready Playwright + TypeScript code following the 3-layer architecture (Page Object → Workflow → Spec).

**MCP Server:** `playwright-test` (browser tools + `generator_setup_page` + `generator_read_log` + `generator_write_test`)

**Hard Rule:** If it was not seen, verified, or confirmed in the browser or existing codebase, do not invent it.

#### Instruction Dependencies (all auto-loaded via `applyTo`)

| Instruction File | Auto-Loads For | Consuming Project Path |
|---|---|---|
| `page-objects.instructions.md` | `pages/**/*.ts` | `.github/instructions/page-objects.instructions.md` |
| `workflows.instructions.md` | `workflows/**/*.ts` | `.github/instructions/workflows.instructions.md` |
| `spec-files.instructions.md` | `tests/**/*.spec.ts` | `.github/instructions/spec-files.instructions.md` |
| `salesforce-stability.instructions.md` | `tests/**/*.ts, pages/**/*.ts, workflows/**/*.ts` | `.github/instructions/salesforce-stability.instructions.md` |
| `test-data.instructions.md` | `test-data/**` | `.github/instructions/test-data.instructions.md` |
| `helper-utilities.instructions.md` | `tests/**/*.ts, workflows/**/*.ts` | `.github/instructions/helper-utilities.instructions.md` |
| `api-tests.instructions.md` | `tests/**/*.ts` | `.github/instructions/api-tests.instructions.md` |

Plus `.github/copilot-instructions.md` (local) for project-specific context.

#### Output

Generated files: page objects, workflows, spec files, CSV test data — only what the scenario requires.

---

### playwright-test-healer

**Purpose:** Debug and fix failing Playwright tests without violating framework rules.

**MCP Server:** `playwright-test` (browser tools + `test_run` + `test_debug` + `test_list`)

#### Common Fix Patterns

| Issue | Correct Fix |
|---|---|
| Stale element | Use getter properties (re-resolved each access) |
| Element not found | Check `c-*` scoping, add ResilientLocator fallbacks |
| Timing failure | Web-first assertions (`expect(locator).toBeVisible()`), spinner wait |
| Combobox failure | 3-attempt retry loop |
| Intermittent timeout | Apply session refresh middleware |
| Cleanup skipped | Wrap toast + cleanup in `try/catch/finally` |
| Environment issue (not code) | `test.fixme()` with explanatory comment |

#### Instruction Dependencies

Same as Generator — all centralized instruction files under `.github/instructions/` apply.
Plus `.github/copilot-instructions.md` (local) for project-specific context.

#### Output

Fixed test files that pass — preserving all framework rules.

---

### AI-test-case-step-generator

**Purpose:** Convert Playwright codegen output (recorded browser interactions) into structured, human-readable manual test cases.

**Tools:** `read`, `edit`, `search` (no MCP or browser dependency)

> This agent is standalone. It does not participate in the Playwright pipeline.

#### Instruction Dependencies

None — this agent is self-contained.

#### Output

Structured manual test cases in markdown table format.

---

## Agent Boundary Summary

| Concern | Planner | Generator | Healer | Manual TC |
|---|:---:|:---:|:---:|:---:|
| Explore live app | **Yes** | **Yes** | **Yes** | — |
| Design test scenarios | **Yes** | — | — | — |
| Write TypeScript code | — | **Yes** | **Yes** | — |
| Create page objects | — | **Yes** | **Yes** | — |
| Create CSV test data | — | **Yes** | — | — |
| Fix failing tests | — | — | **Yes** | — |
| Produce manual test cases | — | — | — | **Yes** |
| Use instruction files | Partial | **All** | **All** | None |
| Require MCP + browser | **Yes** | **Yes** | **Yes** | — |
| **Location** | **Centralized** | **Centralized** | **Centralized** | **Centralized** |
