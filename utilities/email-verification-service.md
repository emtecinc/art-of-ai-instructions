# EmailVerificationService Utility

## When to Use
- Verify emails sent by Salesforce (workflow rules, process builder, flows, triggers)
- Confirm email content, subject, recipients
- Test email notifications and alerts
- Validate email templates and personalization

---

## When NOT to Use
- ❌ For non-Salesforce emails → use inbox provider directly

---

## Import
```typescript
import {
  EmailVerificationService,
  SalesforceEmailProvider,
  MailosaurInboxProvider,
  OutlookInboxProvider,
  type EmailQuery,
  type VerificationResult,
} from 'playwright-custom-core';
```

---

## Required Environment Variables

### For Salesforce (always required)
| Variable | Description |
|---|---|
| `SF_USERNAME` | Automation user |
| `SF_CLIENT_ID` | Connected App ID |
| `SF_PRIVATE_KEY_PATH` | RSA private key path |
| `SF_API_VERSION` | API version (default: `v65.0`) |

### For Mailosaur
| Variable | Description |
|---|---|
| `MAILOSAUR_API_KEY` | Mailosaur API key |
| `MAILOSAUR_SERVER_ID` | Mailosaur server ID |

### For Outlook
| Variable | Description |
|---|---|
| `MS_TENANT_ID` | Azure AD tenant ID |
| `MS_CLIENT_ID` | App registration ID |
| `MS_CERT_THUMBPRINT` | Certificate thumbprint |
| `MS_PRIVATE_KEY_PATH` | Certificate private key path |

---

## Constructor

```typescript
new EmailVerificationService(emailProvider: IEmailProvider, inboxProvider: IInboxProvider)
```

| Parameter | Type | Description |
|---|---|---|
| `emailProvider` | `IEmailProvider` | Source provider (Salesforce) |
| `inboxProvider` | `IInboxProvider` | Destination provider (Mailosaur/Outlook) |

---

## Methods

### `verify(query: EmailQuery, retryConfig?: RetryConfig): Promise<VerificationResult>`
**End-to-end verification**: Fetch from Salesforce and confirm inbox arrival.

```typescript
const verifier = new EmailVerificationService(
  new SalesforceEmailProvider(),
  new MailosaurInboxProvider('your-server-id'),
);

const result = await verifier.verify({
  sentAfter: new Date(Date.now() - 60000), // time at which flow/process/user sends/triggers the email
  address: 'jane@example.com', // Receiver's mail address
  subject: 'Welcome to Acme',
});

// result.allVerified === true when all SF emails matched in inbox
// result.matchedEmails — array of matched pairs
// result.unmatchedEmails — SF emails with no inbox match
```

### `verifyWithSalesforce(query: EmailQuery, retryConfig?: RetryConfig): Promise<EmailRecord[]>`
**Salesforce-only verification**: Confirm Salesforce sent the email (no inbox check).

```typescript
const emails = await verifier.verifyWithSalesforce({
  sentAfter: new Date(Date.now() - 60000), // time at which flow/process/user sends/triggers the email
  address: 'jane@example.com', // Receiver's mail address only
  subject: 'Welcome to Acme',
});
// Returns: EmailRecord[] — throws if none found
```

### `verifyWithMailosaur(query: EmailQuery, retryConfig?: RetryConfig): Promise<InboxEmail[]>`
**Inbox-only verification**: Check Mailosaur directly without querying Salesforce.

```typescript
const emails = await verifier.verifyWithMailosaur({
  sentAfter: new Date(Date.now() - 60000), // time at which flow/process/user sends/triggers the email
  address: 'jane@example.com', // Sender's mail address only
  subject: 'Welcome to Acme',
});
// Returns: InboxEmail[] — throws if none found
```

---

## Types

### EmailQuery
```typescript
interface EmailQuery {
  sentAfter: Date;    // Only emails sent after this timestamp
  address: string;    // Recipient or Sender address
  subject: string;    // Email subject to match
}
```

### RetryConfig
```typescript
interface RetryConfig {
  maxAttempts: number;  // Default: 6
  delayMs: number;      // Default: 10_000
}
```

### VerificationResult
```typescript
interface VerificationResult {
  allVerified: boolean;              // True when all SF emails matched in inbox
  matchedEmails: MatchedEmailPair[]; // Successfully matched pairs
  unmatchedEmails: EmailRecord[];    // SF emails with no inbox match
}
```

### MatchedEmailPair
```typescript
interface MatchedEmailPair {
  salesforceEmail: EmailRecord;
  inboxEmail: InboxEmail;
}
```

### EmailRecord (Salesforce)
```typescript
interface EmailRecord {
  id: string;
  subject: string;
  toAddress: string;
  fromAddress: string;
  textBody?: string;
  htmlBody?: string;
  sentDate: Date;
}
```

### InboxEmail
```typescript
interface InboxEmail {
  id: string;
  subject: string;
  from: string;
  to: string;
  receivedDate: Date;
  textBody?: string;
  htmlBody?: string;
}
```

**Use cases**:
- Email delivery delays → increase `maxAttempts` and `delayMs`
- Fast Salesforce processing → reduce `delayMs`
- Flaky inbox APIs → increase `maxAttempts`

---

## Critical Rules

- ✅ **Capture sentAfter just BEFORE triggering email** — sentAfter time should be acquired when a triggering activity is just about to be invoked to avoid false matches
- ✅ **Use relevant email addresses** — Ensure that sender's & recipient's address is correct and not random
- ✅ **Increase retries for slow delivery** — email may take 10-30 seconds
- ✅ **Verify subject and content** — confirm correct email sent
- ✅ **Verify sender and receiver** — there should be not be any confusion between sender and receiver
- ❌ **Don't capture sentAfter after trigger** — may miss the email
- ❌ **Don't EVER use verifyWithSalesforce and verifyMailosaur in the same test** — use `verify()` instead. Unless the subject of email differs in sent and recieved email. 

---
