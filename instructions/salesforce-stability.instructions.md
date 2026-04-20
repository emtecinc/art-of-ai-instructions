---
applyTo: "tests/**/*.ts, pages/**/*.ts, workflows/**/*.ts"
---

# Salesforce Lightning UI — Stability & Synchronization Rules

ENFORCE these rules STRICTLY across all TypeScript files under `tests/`, `pages/`, and `workflows/` to address real Salesforce production failures.

---

## Synchronization — Wait Priority

| Priority | Pattern | When |
|----------|---------|------|
| 1 | `await expect(locator).toBeVisible()` | Default — web-first assertion |
| 2 | `await page.locator('.slds-spinner').waitFor({ state: 'hidden' })` | After navigation or save |
| 3 | `await locator.waitFor({ state: 'visible' })` | Specific element readiness |
| 4 | `await page.waitForLoadState('networkidle')` | Strictly last resort |

**FORBIDDEN:** `await page.waitForTimeout(...)` — never use hard/static waits.

## Lightning SPA Behavior

- URL changes may precede content rendering — combine URL checks with element visibility
- After navigation/save, wait for spinner: `await page.locator('.slds-spinner').waitFor({ state: 'hidden', timeout: 15000 }).catch(() => {})`
- A record page is "ready" when header title + action buttons (Edit, Delete) are visible

## Toast Handling

Toast messages are transient — they auto-dismiss after a few seconds. Toast verification is best-effort:
- `verifySuccessToast()` in page objects wraps assertion in `try/catch` (warn only, never throw)
- Spec calls `workflow.verifySuccessToast()` in `try/catch/finally` with cleanup in `finally`
- Toast overlay blocking is handled by `addLocatorHandler` in spec `beforeEach`
- See `page-objects.instructions.md` and `spec-files.instructions.md` for patterns

## Shadow DOM

- Playwright auto-pierces Shadow DOM — use standard locator APIs
- Never guess component names — inspect DOM for exact `c-*` wrapper
- Use the nearest LWC component root for scoping

## Stale Element Prevention

- Never cache locators across page refreshes — use getter properties
- After Save/Edit, re-query all elements

## Dynamic Field IDs

- Salesforce generates dynamic IDs (`input-125`) — **never use ID selectors**
- Use `getByLabel()`, `getByRole()`, or stable attributes (`name`, `placeholder`, `aria-label`)

## Element Interception

- `scrollIntoViewIfNeeded()` before clicking off-screen elements
- Ensure spinners are gone before interacting
- Use `.click({ force: true })` only as documented last resort
- **Duplicate detection + toast overlay** handlers MUST be registered via `addLocatorHandler` in spec `beforeEach` — see `spec-files.instructions.md`

## Combobox Interactions

Combobox selection uses a 3-attempt retry pattern with text/label-based selection (never index). The authoritative rules and code pattern are in `page-objects.instructions.md` — follow them exactly.

**Always select by visible text/label — never by index.** Applies to comboboxes, picklists, radio buttons, and any multi-option component on both parent and child/related forms.

## URL Assertions

```typescript
// ✅ Includes object name
await expect(page).toHaveURL(/\/lightning\/r\/Account\/001[a-zA-Z0-9]+\/view/, { timeout: 30_000 });
```

| Object | ID Prefix |
|--------|-----------|
| Account | `001` |
| Contact | `003` |
| Opportunity | `006` |
| Lead | `00Q` |
| Case | `500` |

## Related Lists

```typescript
page.getByRole('article', { name: 'Related Accounts' }); // ✅ accessible name
```

## Data Tables

```typescript
page.getByRole('row', { name: /Account Name/ });  // ✅ Never use nth-child
```

## Common Failure Patterns

| Failure | Root Cause | Fix |
|---------|-----------|-----|
| `element not attached to DOM` | Cached locator after refresh | Use getter properties |
| Strict mode violation | Unscoped locator / multiple matches | Scope to dialog/component, `exact: true` |
| Click intercepted | Toast/spinner covering target | `addLocatorHandler` in spec `beforeEach` (MANDATORY) + spinner wait |
| Timeout on combobox | Dropdown closed by overlay | 3-attempt retry loop |
| Partial text in input | `pressSequentially` + focus theft | Use `fill()` |
| Toast not found / toast timeout | Toast auto-dismissed before assertion | `try/catch/finally` in spec, best-effort verification (see `spec-files.instructions.md`) |

## Enterprise Execution

- Support `DEV`, `QA`, `UAT` via `process.env.BASE_URL`
- Tags MUST follow the EXACT format defined in `spec-files.instructions.md` — Jira key first, then test type tags, declared only inside `test()` as an array
- Max 1 retry for infrastructure blips
- Never commit credentials — use env vars
- Config: `screenshot: 'only-on-failure'`, `video: 'retain-on-failure'`, `trace: 'retain-on-failure'`
