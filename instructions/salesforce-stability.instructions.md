---
applyTo: "tests/**/*.ts"
---

# Salesforce Lightning UI — Stability & Synchronization Rules

These rules apply to ALL TypeScript files under `tests/` and address real Salesforce production failures.

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

## Toast Transience (CRITICAL)

Salesforce Lightning success toasts are **transient UI elements** that auto-dismiss after a few seconds.

**Rules:**
- Toast verification MUST happen **immediately** after the save/create action — delays risk the toast disappearing
- Toast verification MUST be wrapped in `try/catch` — it is **best-effort**, not guaranteed
- Toast verification failure MUST NOT block subsequent operations (record identity, cleanup registration)
- Toast is an ephemeral confirmation signal, NOT a gate for downstream logic
- This applies to ALL creation patterns: full-page, modal, inline, quick action, embedded/LWC

**Implementation:** Toast verification lives in page object methods with internal try/catch that warns but does not throw. See `page-objects.instructions.md` for the authoritative pattern.

**Cross-references:**
- `page-objects.instructions.md` — `verifySuccessToast()` implementation pattern
- `workflows.instructions.md` — toast step placement in creation flows
- `spec-files.instructions.md` — cleanup registration MUST NOT depend on toast success

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

## Combobox Interactions

Combobox selection uses a 3-attempt retry pattern with text/label-based selection (never index). The authoritative rules and code pattern are in `page-objects.instructions.md` — follow them exactly.

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
| Click intercepted | Spinner covering target | Wait for spinner to be hidden before interacting |
| Timeout on combobox | Dropdown closed by overlay | 3-attempt retry loop |
| Partial text in input | `pressSequentially` + focus theft | Use `fill()` |
| Toast not found / toast timeout | Toast auto-dismissed before assertion | Wrap in try/catch, treat as best-effort (see Toast Transience above) |

## Enterprise Execution

- Support `DEV`, `QA`, `UAT` via `process.env.BASE_URL`
- Tags: `@smoke`, `@regression`, `@sanity`
- Max 1 retry for infrastructure blips
- Never commit credentials — use env vars
- Config: `screenshot: 'only-on-failure'`, `video: 'retain-on-failure'`, `trace: 'retain-on-failure'`

```
