---
applyTo: tests/**/*.ts, pages/**/*.ts, workflows/**/*.ts
---
# Salesforce Lightning UI — Stability & Synchronization Rules
ENFORCE these rules STRICTLY to address real Salesforce production failures.

## Synchronization — Wait Priority
| Priority | Pattern | When |
|-|-|-|
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
- Salesforce generates dynamic IDs (`input-125`) — **NEVER use ID selectors**
- Use `getByLabel()`, `getByRole()`, or stable attributes (`name`, `placeholder`, `aria-label`)

## Element Interception
- `scrollIntoViewIfNeeded()` before clicking off-screen elements
- Ensure spinners are gone before interacting
- Use `.click({ force: true })` only as documented last resort
- **Duplicate detection + toast overlay** handlers MUST be registered via `addLocatorHandler` in spec `beforeEach` — see `spec-files.instructions.md`

## Combobox Interactions
Combobox selection ALWAYS uses a 3-attempt retry pattern with text/label-based selection (NEVER index). The authoritative rules and code pattern are present in `page-objects.instructions.md` — follow them EXACTLY.

**Always select by visible text/label — never by index.** Applies to comboboxes, picklists, radio buttons, and any multi-option component on both parent and child/related forms.

## URL Assertions
```typescript
// Includes object name
await expect(page).toHaveURL(/\/lightning\/r\/Account\/001[a-zA-Z0-9]+\/view/, { timeout: 30_000 });
```

## Related Lists
```typescript
page.getByRole('article', { name: 'Related Accounts' }); // Accessible name
```

## Data Tables
```typescript
page.getByRole('row', { name: /Account Name/ });  // NEVER use nth-child
```

## Common Failure Patterns
| Failure | Root Cause | Fix |
|-|-|-|
| `element not attached to DOM` | Cached locator after refresh | Use getter properties |
| Strict mode violation | Unscoped locator / multiple matches | Scope to dialog/component, `exact: true` |
| Click intercepted | Toast/spinner covering target | `addLocatorHandler` in spec `beforeEach` (MANDATORY) + spinner wait |
| Timeout on combobox | Dropdown closed by overlay | 3-attempt retry loop |
| Partial text in input | `pressSequentially` + focus theft | Use `fill()` |
| Toast not found / toast timeout | Toast auto-dismissed before assertion | `try/catch/finally` in spec, best-effort verification (see `spec-files.instructions.md`) |

## Pop-up Handlers
```typescript
// Generalized handler for Salesforce Feature Discovery / In-App Guidance docked composers
await page.addLocatorHandler(
    page.locator('div[role="dialog"].slds-docked-composer')
    .filter({
      has: page.locator('.forceFeatureDiscoveryDockedContent')
    }),
  async (dialog) => {
    try {
      // Try Close button first
      const closeButton = dialog.getByRole('button', { name: 'Close', exact: true })
        .or(dialog.locator('button.closeButton'))
        .or(dialog.locator('button[title="Close"]'))
        .or(dialog.locator('.closeButton'));

      if (await closeButton.count() > 0) {
        await closeButton.first().click({ timeout: 10_000 });
        return;
      }

      // Fallback: Minimize button
      const minimizeButton = dialog.getByRole('button', { name: 'Minimize', exact: true })
        .or(dialog.locator('button.minButton'))
        .or(dialog.locator('.minButton'));

      if (await minimizeButton.count() > 0) {
        await minimizeButton.first().click({ timeout: 10_000 });
        return;
      }

      // Last resort: Any button with Close/Minimize text
      const fallbackButton = dialog.locator('button')
        .filter({ hasText: /Close|Minimize/i })
        .first();

      if (await fallbackButton.count() > 0) {
        await fallbackButton.click({ timeout: 8_000 }).catch(() => {});
      }
    } catch (error) {
      console.warn(`Failed to close feature discovery composer:`, error instanceof Error ? error.message : error);
    }
  },{ noWaitAfter: true}
);

// Handler for persistent floating panels with focus-stealing behavior
await page.addLocatorHandler(
    page.locator('div[role="dialog"].uiPanel--floatingPanel')
    .filter({
      has: page.locator('one-floating-panel-content')
    })
    .filter({
      has: page.locator('h2, .slds-popover_prompt__heading')
      .filter({ hasText: /Try|Try the new Salesforce Setup|Salesforce Go|new Salesforce Setup/i })
    }),
  async (floatingPanel) => {
    try {
      const dismissButton = floatingPanel.getByRole('button', { name: 'Dismiss', exact: true })
        .or(floatingPanel.locator('lightning-button.dismiss button'))
        .or(floatingPanel.locator('button').filter({ hasText: /Dismiss/i }));

      if (await dismissButton.count() > 0) {
        await dismissButton.first().click({ timeout: 12_000 });
        return;
      }

      // Fallback: Any close-like button or snooze
      const closeOrSnooze = floatingPanel.locator('button')
        .filter({ hasText: /Dismiss|Snooze|Close|Pause/i })
        .first();

      if (await closeOrSnooze.count() > 0) {
        await closeOrSnooze.click({ timeout: 10_000 }).catch(() => {});
        return;
      }

      const snoozeButton = floatingPanel.locator('lightning-button-menu.snooze-menu-position button');
      if (await snoozeButton.count() > 0) {
        await snoozeButton.first().click({ timeout: 8_000 }).catch(() => {});
        return;
      }
    } catch (error) {
      console.warn(`Failed to dismiss floating panel:`, 
        error instanceof Error ? error.message : error);
    }
  },{ noWaitAfter: true }
);
```
## Enterprise Execution
- Support `DEV`, `QA`, `UAT` via `process.env.BASE_URL`
- Tags MUST follow the EXACT format defined in `spec-files.instructions.md`. Apply them correctly and consistently in EVERY spec file.
- Max 1 retry for infrastructure blips
- Never commit credentials — use env vars