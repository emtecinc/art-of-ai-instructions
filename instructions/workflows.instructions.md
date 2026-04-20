---
applyTo: "workflows/**/*.ts"
---

# Workflow Rules

## Class Structure

Every Workflow MUST:
- Extend `BaseWorkflow` from `playwright-custom-core`
- Declare `readonly workflowName: string`
- Accept `(page: Page, baseUrl?: string)` in the constructor
- Instantiate **only the page objects needed** for the specific scenario

```typescript
import { Page, test } from '@playwright/test';
import { BaseWorkflow } from 'playwright-custom-core';
import { OpportunityListPage } from '../../pages/opportunities/opportunity-list-page';
import { OpportunityCreationPage } from '../../pages/opportunities/opportunity-creation-page';

export class CreateOpportunityWorkflow extends BaseWorkflow {
  readonly workflowName = 'CreateOpportunityWorkflow';

  private listPage: OpportunityListPage;
  private creationPage: OpportunityCreationPage;

  constructor(page: Page, baseUrl?: string) {
    super(page);
    const url = baseUrl || process.env.BASE_URL || '';
    this.listPage = new OpportunityListPage(page, url);
    this.creationPage = new OpportunityCreationPage(page, url);
  }

  protected override async testStep(description: string, action: () => Promise<void>): Promise<void> {
    await test.step(`${this.workflowName}: ${description}`, action);
  }
}
```

## Demand-Driven Page Imports (CRITICAL)

Import and instantiate **only the page objects needed** for the workflow's scenario. Don't import all pages for an object.

```typescript
// ✅ VALID — Only uses list + creation pages
import { OpportunityListPage } from '../../pages/opportunities/opportunity-list-page';
import { OpportunityCreationPage } from '../../pages/opportunities/opportunity-creation-page';

// ❌ INVALID — Imports pages not used in this workflow
import { OpportunityDetailPage } from '...';
import { OpportunityRelatedPage } from '...';
```

## SF-Basepage Usage

Every workflow that needs navigation MUST use `SalesforcePage` from `pages/SF-Basepage/sf-page.ts`:

```typescript
import { SalesforcePage } from '../../pages/SF-Basepage/sf-page';

// In constructor:
this.sfPage = new SalesforcePage(page, url);
```

Navigation methods (`navigateToBaseUrl`, `closeAllPrimaryTabs`, `navigateToAppViaAppLauncher`) are exposed as workflow methods that wrap `this.sfPage.*` calls inside `this.testStep()`.

**All navigation goes through SF-Basepage.** Do NOT duplicate app launcher/search/tab logic in workflows or object pages. See `page-objects.instructions.md` for SF-Basepage rules and interface.

## One Workflow Per Scenario

Each unique business scenario gets its own Workflow class:

- `create-account-required-fields-workflow.ts`
- `edit-account-workflow.ts`
- `display-details-tab-workflow.ts`

Multiple workflows can use the SAME Page Object. Organized in `workflows/<object>/`.

## Every Method Uses `this.testStep()`

```typescript
async navigateToAccounts(): Promise<void> {
  await this.testStep('Navigate to Accounts list page', async () => {
     await this.sfPage.navigateToAppViaAppLauncher('Accounts');
    // Capture screenshot after navigation — critical step
    await this.sfPage.captureScreenshot(this['page'], test.info(), 'navigate-to-accounts.png');
  });
}
```

## High-Level Business Methods (CRITICAL)

Workflows expose **high-level business methods** that the spec calls. Each method internally uses `this.testStep()` for granular actions. Specs MUST NOT call individual workflow steps like `clickNewButton()` or `fillField()`.

```typescript
// ✅ CORRECT — Workflow exposes high-level methods, spec calls them directly
async createEntity(data: EntityData): Promise<void> {
  await this.testStep('Navigate to Entities via App Launcher', async () => {
    await this.sfPage.navigateToAppViaAppLauncher('Entities');
    await this.sfPage.captureScreenshot(this['page'], test.info(), 'navigate-to-entities.png');
  });

  await this.testStep('Click New button', async () => {
    await this.listPage.clickNewButton();
  });

  await this.testStep('Fill entity name', async () => {
    await this.creationPage.fillEntityName(data.name);
  });

  await this.testStep('Select entity type', async () => {
    await this.creationPage.selectType(data.type);
  });

  await this.testStep('Click Save', async () => {
    await this.creationPage.clickSave();
    await this.sfPage.captureScreenshot(this['page'], test.info(), 'click-saved-entity.png');
  });
}

async verifyEntityCreated(data: EntityData): Promise<void> {
  await this.testStep('Verify entity name on detail page', async () => {
    await this.detailPage.verifyEntityName(data.name);
    await this.sfPage.captureScreenshot(this['page'], test.info(), 'verify-entity-created.png');
  });
}
```

```typescript
// ❌ WRONG — Spec should NOT call granular steps
// In spec: await workflow.clickNewButton();
// In spec: await workflow.fillEntityName(data.name);
// In spec: await workflow.clickSave();
```

**Separate Act vs Assert methods** — creation and verification MUST be separate workflow methods. The spec inserts cleanup between them.

## Toast Verification

Expose `verifySuccessToast()` as a **separate workflow method** — do NOT embed it inside create methods. The spec calls it in `try/catch/finally` with cleanup registration.

```typescript
async verifySuccessToast(expectedText: string): Promise<void> {
  await this.testStep('Verify success toast', async () => {
    await this.creationPage.verifySuccessToast(expectedText);
    await this.sfPage.captureScreenshot(this['page'], test.info(), 'verify-success-toast.png');
  });
}
```

## Data Interfaces

Export typed data interfaces for workflow parameters:

```typescript
export interface AccountData {
  name: string;
  type?: string;
  industry?: string;
}
```

## Strict Boundaries

| ✅ Workflow MUST ALWAYS | ❌ Workflow MUST NEVER |
|----------------|---------------------|
| Call Page Object methods (including verification methods) | Contain locators or `page.locator()` |
| Accept typed data from spec | Import `expect` from `@playwright/test` |
| Use `this.testStep()` for logging | Contain `expect()` calls directly |
| Define data interfaces | Directly access DOM elements |
| Use multiple page objects | Embed toast verification inside create methods |
```
