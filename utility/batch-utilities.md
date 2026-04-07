# Batch Execution Utilities

## When to Use
- Execute Apex batch classes
- Wait for batch completion before assertions
- Monitor batch progress (items processed, errors)
- Orchestrate data setup/cleanup via batches

---

## When NOT to Use
- ❌ For synchronous operations → use REST API directly
- ❌ For UI-based batch triggers → use Page Objects

---

## Import
```typescript
import {
  executeBatch,
  runBatch, // Convenience wrapper
  triggerBatch,
  getBatchStatus,
  waitForBatchCompletion,
  type BatchConfig,
  type BatchExecutionResult,
} from 'playwright-custom-core';
```

---

## Components

### 1. BatchExecutor (`executeBatch`)
High-level orchestrator: authenticate → trigger → wait → return result.

### 2. BatchRunner (`triggerBatch`)
Single responsibility: trigger batch via `executeAnonymous`.

### 3. BatchStatus (`getBatchStatus`)
Single responsibility: query `AsyncApexJob` for status.

### 4. BatchWaiter (`waitForBatchCompletion`)
Single responsibility: poll until completion or failure.

---

## High-Level API

### `runBatch(batchClassName: string): Promise<BatchExecutionResult>`
Convenience wrapper with default settings. ALWAYS use this unless BatchConfig is explicitly stated.

```typescript
const result = await runBatch('DataSeedBatch');
// Same as executeBatch({ batchClassName: 'DataSeedBatch' })
```

### `executeBatch(config: BatchConfig): Promise<BatchExecutionResult>`
Complete workflow: trigger batch and wait for completion.

**Usage**:
```typescript
const result = await executeBatch({
  batchClassName: 'AccountCleanupBatch',
  batchSize: 200,
  pollingInterval: 5000,  // 5 seconds
  maxPollingAttempts: 120, // 10 minutes max (120 * 5s)
});

console.log(`Processed ${result.itemsProcessed}/${result.totalItems} items`);
console.log(`Errors: ${result.errors}`);
```

---

## Low-Level APIs

### `triggerBatch(credentials, batchClassName, batchSize?): Promise<void>`
Execute batch via Tooling API `executeAnonymous`.

```typescript
import { SalesforceConnection } from 'playwright-custom-core';

const conn = SalesforceConnection.getInstance();
const credentials = await conn.getConnection();

await triggerBatch(credentials, 'AccountCleanupBatch', 200);
// Batch triggered — does NOT wait for completion
```

### `getBatchStatus(credentials, batchClassName): Promise<BatchJob>`
Query latest batch job status.

```typescript
const job = await getBatchStatus(credentials, 'AccountCleanupBatch');
console.log(`Status: ${job.Status}`);
console.log(`Progress: ${job.JobItemsProcessed}/${job.TotalJobItems}`);
```

### `waitForBatchCompletion(credentials, batchClassName, pollingInterval?, maxAttempts?): Promise<BatchExecutionResult>`
Poll until batch completes or fails.

```typescript
const result = await waitForBatchCompletion(
  credentials,
  'AccountCleanupBatch',
  5000,  // Poll every 5 seconds
  120,   // Max 120 attempts (10 minutes)
);
```

---

## Rules

- ✅ **Use runBatch() for for standard cases** — minimal configuration
- ✅ **Adjust polling based on batch size, ONLY when necessary ** — larger batches need more time
- ❌ **Don't use for synchronous operations** — batches are async by design
- ❌ **Don't poll too frequently** — respect Salesforce API limits
- ❌ **Don't use short timeouts** — batches can take minutes
