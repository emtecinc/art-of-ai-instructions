---
applyTo: "pages/**/*.ts"
---

# Page Object Rules

## Class Structure

Every Page Object MUST:
- Extend `BasePage` from `playwright-custom-core`
- Declare `readonly pageName: string` and `protected readonly relativeUrl: string`
- Accept `(page: Page, baseUrl?: string)` in the constructor

```typescript
import { Page, expect } from '@playwright/test';
import { BasePage, ResilientLocator } from 'playwright-custom-core';

export class EntityPage extends BasePage {
  readonly pageName = 'EntityPage';
  protected readonly relativeUrl = '/lightning/o/Entity/list';

  constructor(page: Page, baseUrl?: string) {
    super(page, baseUrl || process.env.BASE_URL || '');
  }
}
```

## Demand-Driven Page Creation (CRITICAL)

**Create page files ONLY when a test scenario requires them.** Do NOT pre-create all page types for an object.

| Rule | Detail |
|------|--------|
| Create on demand | Only create a page file when a test step requires actions from that page |
| No speculative creation | Never create `<object>-edit-page.ts` or `<object>-related-page.ts` unless a test uses them |
| Extend existing | If a page file exists, extend it with new methods — do NOT rewrite |
| Single test needs | If a test only needs list + creation pages, create only those two |

### When to Create a Separate Page File

- The URL/route changes (list → form → detail)
- The DOM structure is fundamentally different
- Consolidating would create a file > 500 lines

### Page File Naming

**Pattern:** `<object>-<purpose>-page.ts` in `pages/<object>/`

| Purpose | File Name |
|---------|-----------|
| List view | `<Object>-list-page.ts` |
| Record creation | `<Object>-creation-page.ts` |
| Record detail | `<Object>-detail-page.ts` |
| Record edit | `<Object>-edit-page.ts` |
| Related records | `<Object>-related-page.ts` |

## SF-Basepage — Reuse Before Creating (CRITICAL)

Before creating any Salesforce-generic action in an object page, **check `pages/SF-Basepage/sf-page.ts` first**.

**What belongs in SF-Basepage** — Generic Salesforce UI actions with stable DOM across all orgs:
- `navigateToBaseUrl()` — navigate to Salesforce home
- `closeAllPrimaryTabs()` — close all open console tabs before each test
- `navigateToAppViaAppLauncher(appName)` — open App Launcher → search → select
- Global Search, User menu, and any SF-wide actions stable across orgs

If a generic method is missing, **add it to `sf-page.ts`** before creating object-specific alternatives.

**What does NOT belong** — Object-specific forms/lists/details, record-specific actions, custom LWC

### SF-Basepage Core Interface

The following methods are expected in `sf-page.ts` (created on demand when first needed):

```typescript
export class SalesforcePage extends BasePage {
  readonly pageName = 'SalesforcePage';
  protected readonly relativeUrl = '/lightning/page/home';

  // Navigate to Salesforce home page
  async navigateToBaseUrl(): Promise<void> { ... }

  // Close all open console/primary tabs
  async closeAllPrimaryTabs(): Promise<void> { ... }

  // Open App Launcher → search → click matching item
  async navigateToAppViaAppLauncher(appName: string): Promise<void> { ... }
}
```

If `sf-page.ts` is empty or missing a method, **create or add it** following the patterns above before using it in workflows.

### Incremental Growth Rule

SF-Basepage grows **incrementally**. Add methods only when a test needs them:

1. Test needs a generic SF action → Check if method exists in `sf-page.ts`
2. Exists → Reuse via workflow
3. Doesn't exist → Add to `sf-page.ts` (if generic) or object page (if object-specific)

### Navigation via App Launcher

All object navigation uses App Launcher through `SalesforcePage` methods. Workflow calls `sfPage.navigateToAppViaAppLauncher('ObjectName')` — page objects do not handle top-level navigation.

## Locator Rules

All locators are **private getter properties** returning `ResilientLocator` with exactly **3 fallback strategies**:

```typescript
private get accountNameInput() {
  return new ResilientLocator(this['page'], [
    (p) => p.getByRole('textbox', { name: 'Account Name', exact: true }),
    (p) => p.locator('[data-testid="account-name-input"]'),
    (p) => p.locator('input[name*="Name"]'),
  ]);
}
```

### LWC Component Scoping

Inspect DOM first for a stable `c-*` root. If present, scope all 3 strategies inside it.

```typescript
private get nameInput() {
  return new ResilientLocator(this['page'], [
    (p) => p.locator('c-submit-feedback').getByRole('textbox', { name: 'Name' }),
    (p) => p.locator('c-submit-feedback').getByPlaceholder('Enter name'),
    (p) => p.locator('c-submit-feedback input[name="name"]'),
  ]);
}
```

**Exception:** Overlays (modals, dropdowns) render globally — do NOT scope to `c-*`.

### Locator Priority Order

1. `getByRole` → 2. `getByLabel` → 3. `getByPlaceholder` → 4. `getByText` → 5. `data-testid`/`data-id` → 6. Minimal CSS

### Forbidden Selectors — NEVER use:

`id` attributes, `class`/`slds-*`, XPath, `nth-child`/index-based, deep CSS chains.

### Select by Text/Label — NEVER by Index (CRITICAL)

For **combobox, picklist, radio button, dropdown**, and any multi-option component — always select the item by its visible **text or label**. This applies to both parent and child/related record forms.

```typescript
// ✅ CORRECT — Select by text/label
await this.selectComboboxOption('Stage', 'Prospecting');
await page.getByRole('option', { name: 'Closed Won' }).click();
await page.getByLabel('Active').selectOption({ label: 'Yes' });
await page.getByRole('radio', { name: 'Business' }).check();

// ❌ WRONG — Never select by index
await combobox.selectOption({ index: 2 });
await options.nth(3).click();
await page.getByRole('radio').nth(0).check();
```

### exact:true for Ambiguous Labels

```typescript
(p) => p.getByRole('textbox', { name: 'Email', exact: true })
```

### Avoiding Strict Mode Violations (CRITICAL)

**Strict Mode Violation** occurs when a locator matches multiple elements. Playwright rejects ambiguous locators to prevent false positives.

#### Common Causes:

1. **Partial Text Matching** — Regex or `:has-text()` without exact match
2. **Missing `exact: true`** — For getByRole/getByLabel/getByText
3. **Broad Selectors** — CSS/attribute selectors that match multiple elements

#### Example Failure:

```typescript
// ❌ BAD — Matches both "Opportunities" AND "Opportunities Recently Viewed"
private get listViewHeading() {
  return new ResilientLocator(this['page'], [
    (p) => p.getByRole('heading', { name: /Opportunities/i }),  // Partial match!
    (p) => p.locator('h1:has-text("Opportunities")'),           // Partial match!
    (p) => p.locator('h1'),                                     // Too broad!
  ]);
}
```

**Error:** `strict mode violation: resolved to 2 elements`

#### The Fix — All 3 Strategies Must Be Specific:

```typescript
// ✅ GOOD — All strategies match exactly one element
private get listViewHeading() {
  return new ResilientLocator(this['page'], [
    (p) => p.getByRole('heading', { name: 'Opportunities', exact: true }),
    (p) => p.locator('h1').filter({ hasText: /^Opportunities$/ }),
    (p) => p.locator('[data-aura-class*="forceListViewManager"] h1:has-text("Opportunities")').first(),
  ]);
}
```

**Key Principles:**
- **Strategy 1:** Always use `exact: true` for getByRole/getByLabel/getByText when possible
- **Strategy 2:** Use `.filter({ hasText: /^exactText$/ })` for exact regex matching
- **Strategy 3:** Add `.first()` or parent scoping to limit matches
```

### Dialog-Scoped Locators

```typescript
const dialog = this['page'].getByRole('dialog', { name: 'New Contact' });
await dialog.getByRole('textbox', { name: 'Email', exact: true }).fill(email);
```

## Action Methods

All action methods are `public async` with `try/catch`:

```typescript
async fillAccountName(name: string): Promise<void> {
  try {
    await this.accountNameInput.getLocator().fill(name);
  } catch (error) {
    console.error('Failed to fill Account Name');
    throw error;
  }
}
```

### Save/Create Methods Must Wait for Completion

Save/create methods MUST handle the full post-save page-level sequence:
1. Click the save/create button
2. Wait for spinner to clear
3. **Immediately** attempt toast verification (the toast disappears quickly)
4. Toast verification MUST be wrapped in `try/catch` — failure to verify toast MUST NOT block subsequent operations

```typescript
async clickSave(): Promise<void> {
  try {
    await this.saveButton.getLocator().click();
    await this['page'].locator('.slds-spinner').waitFor({ state: 'hidden', timeout: 15000 }).catch(() => {});
  } catch (error) {
    console.error('Failed to click Save');
    throw error;
  }
}

async verifySuccessToast(expectedText?: string): Promise<void> {
  try {
    const toast = this['page'].locator('.toastMessage');
    await expect(toast).toBeVisible({ timeout: 10000 });
    if (expectedText) {
      await expect(toast).toContainText(expectedText, { timeout: 5000 });
    }
  } catch {
    console.warn('Toast verification failed — toast may have disappeared');
    // Toast is transient — do NOT throw. See salesforce-stability.instructions.md.
  }
}
```

**Key rules for save/create page methods:**
- Toast verification is **best-effort** — it MUST NOT throw on failure
- The `verifySuccessToast()` method MUST swallow exceptions (warn only)
- This pattern applies to ALL creation flows: full-page, modal, inline, quick action, embedded/LWC
- Workflows call `verifySuccessToast()` immediately after save — see `workflows.instructions.md`

## Verification Methods (Assertions Live HERE)

All `expect()` calls belong in Page Object verification methods only. Keep methods generic for reuse:

```typescript
async verifyHeadingContains(expectedText: string): Promise<void> {
  try {
    const heading = this['page'].getByRole('heading', { name: new RegExp(expectedText) });
    await expect(heading).toBeVisible({ timeout: 15000 });
  } catch (error) {
    console.error(`Failed to verify heading: ${expectedText}`);
    throw error;
  }
}
```

### Lightning Combobox Assertion

```typescript
await expect(comboboxLocator).toContainText(expectedValue);  // ✅ (NOT toHaveValue)
```

## Combobox Interaction Pattern (Retry Loop)

Always select by **option text** — never by index or position.

```typescript
async selectComboboxOption(comboboxName: string, optionText: string): Promise<void> {
  const combobox = this['page'].getByRole('combobox', { name: comboboxName });
  const option = this['page'].getByRole('option', { name: optionText });
  for (let attempt = 0; attempt < 3; attempt++) {
    try {
      await combobox.click({ timeout: 10_000 });
      await expect(option).toBeVisible({ timeout: 5_000 });
      await option.click();
      return;
    } catch { /* Dropdown closed by overlay — retry */ }
  }
  await combobox.click({ timeout: 10_000 });
  await expect(option).toBeVisible({ timeout: 5_000 });
  await option.click();
}
```

## Anti-Patterns

- Never call methods directly on a `ResilientLocator` instance — always call `.getLocator()` first: `resilientLocator.getLocator().click()`
- Never cache locators across page transitions — use getters
- Never use `pressSequentially` — use `fill()`
- Never use `waitForTimeout()` — see `salesforce-stability.instructions.md`
- Never duplicate SF-Basepage actions in object pages
- Never put object-specific logic in SF-Basepage
- Never throw from toast verification methods — toast is transient and best-effort

```
