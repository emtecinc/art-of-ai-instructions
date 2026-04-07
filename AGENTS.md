# Agent Registry

This file documents all agents in the system — their purpose, architecture placement, boundaries, and instruction dependencies.

---

## Architecture: Centralized vs Local

| Agent | Centralized? | Location | Why |
|---|---|---|---|
| `AI-test-case-step-generator` | **Yes** | `agents/manual-test-step-generator.agent.md` | Text-only, no MCP/browser dependency |
| `playwright-test-planner` | No | Consuming project `.github/agents/` | Requires MCP server + live browser |
| `playwright-test-generator` | No | Consuming project `.github/agents/` | Requires MCP server + live browser |
| `playwright-test-healer` | No | Consuming project `.github/agents/` | Requires MCP server + live browser |

### Pipeline

```
Planner ──(plan)──► Generator ──(code)──► Healer
                                              │
                         fixes code following same rules
```

The **Manual Test Step Generator** is standalone — it does not participate in the Playwright pipeline.

### How Local Agents Consume Centralized Instructions

Local Playwright agents are **lightweight orchestrators**. They define:
- Tool configuration (MCP server, browser tools)
- Workflow sequence (what to do, in what order)
- Execution-specific behavior (browser-first verification, save locations)

**All framework rules, coding patterns, and architectural constraints live in centralized instruction files under `ai-instructions/instructions/`.** Local agents MUST:

1. Follow auto-loaded instruction rules without overriding them
2. Use `.github/copilot-instructions.md` for project-specific context only
3. Use `.github/local-override.md` for project-specific exceptions only
4. Never duplicate centralized instruction content

---

## Centralized Agent: AI-test-case-step-generator

**File:** `agents/manual-test-step-generator.agent.md`
**Purpose:** Convert Playwright codegen output (recorded browser interactions) into structured, human-readable manual test cases.

**Model:** Default
**Tools:** `read`, `edit`, `search`

> This agent is standalone. It does not participate in the Playwright pipeline.

### MUST Do

- Parse all `test(...)` blocks from codegen output
- Convert each block into a structured manual test case (title, preconditions, steps table, test data, postconditions)
- Derive element labels from `aria-label`, `placeholder`, `text`, or `id` — never expose raw selectors
- Refer to dropdown options and radio buttons by visible label/text, never by index
- Group consecutive `fill` calls on the same form into one step
- Move hardcoded values to the Test Data section
- Map `beforeEach` → Preconditions, `afterEach` → Postconditions
- Keep each step atomic (one user action, ≤ 20 words, active voice)

### MUST NOT Do

- Write automation code
- Suggest automation fixes
- Include raw CSS/XPath selectors in steps
- Reference elements by index position
- Include non-user steps (`waitForTimeout`, `console.log`, config setup)

### Instruction Dependencies

None — this agent is self-contained.

### Output

Structured manual test cases in markdown table format.

---

## Local Agents — Reference Documentation

The following agents are LOCAL ONLY. They live in `.github/agents/` in each consuming project.
This section documents their boundaries and instruction dependencies for reference. The actual agent files are maintained locally.

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
| `instructions/helper-utilities.instructions.md` | `ai-instructions/instructions/helper-utilities.instructions.md` | Identify available utilities for test implementation |
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
| `page-objects.instructions.md` | `pages/**/*.ts` | `ai-instructions/instructions/page-objects.instructions.md` |
| `workflows.instructions.md` | `workflows/**/*.ts` | `ai-instructions/instructions/workflows.instructions.md` |
| `spec-files.instructions.md` | `tests/**/*.spec.ts` | `ai-instructions/instructions/spec-files.instructions.md` |
| `salesforce-stability.instructions.md` | `tests/**/*.ts` | `ai-instructions/instructions/salesforce-stability.instructions.md` |
| `test-data.instructions.md` | `test-data/**` | `ai-instructions/instructions/test-data.instructions.md` |
| `helper-utilities.instructions.md` | `tests/**/*.ts` | `ai-instructions/instructions/helper-utilities.instructions.md` |

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
| Environment issue (not code) | `test.fixme()` with explanatory comment |

#### Instruction Dependencies

Same as Generator — all centralized instruction files under `ai-instructions/instructions/` apply.
Plus `.github/copilot-instructions.md` (local) for project-specific context.

#### Output

Fixed test files that pass — preserving all framework rules.

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
| **Location** | **Local** | **Local** | **Local** | **Centralized** |
