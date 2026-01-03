# Feature Guide - AI Agent Reference

Explains SESMailEngine features, use cases, and how things work. Use this to answer "how do I..." and "why does..." questions.

---

## System Overview

SESMailEngine is a serverless email infrastructure that:
1. Receives email requests via EventBridge
2. Loads templates from S3
3. Sends emails via SES
4. Tracks delivery status in DynamoDB
5. Handles bounces/complaints automatically
6. Protects SES reputation with bounce rate limits

**Flow:**
```
App → EventBridge → Lambda → SES → Recipient
                      ↓
                  DynamoDB (tracking)
                      ↑
              SNS ← SES (feedback)
```

---

## Sending Emails

### Basic Email Request

```json
{
  "source": "my.application",
  "detail-type": "Email Request",
  "detail": {
    "to": "recipient@example.com",
    "templateName": "welcome",
    "templateData": {
      "userName": "John",
      "companyName": "Acme Corp"
    }
  }
}
```

### With All Options

```json
{
  "source": "my.application",
  "detail-type": "Email Request",
  "detail": {
    "emailId": "custom-id-123",
    "to": "recipient@example.com",
    "subject": "Override subject",
    "templateName": "welcome",
    "templateData": {
      "userName": "John",
      "companyName": "Acme Corp"
    },
    "senderOverride": {
      "name": "Custom Sender",
      "email": "custom@example.com"
    },
    "metadata": {
      "campaignId": "summer-2024",
      "userId": "user-456"
    }
  }
}
```

---

## Sender Priority Chain

Both email and name are resolved independently through this priority chain:

| Priority | Source | When to Use |
|----------|--------|-------------|
| 1 (highest) | `senderOverride` in event | Per-email exceptions |
| 2 | Template `metadata.json` | Department/template defaults |
| 3 (lowest) | CloudFormation `DefaultSenderEmail/Name` | Global fallback |

### Example Scenarios

**Scenario 1: Department-based senders**
```
Template: invoice/metadata.json
  "senderEmail": "billing@company.com"
  "senderName": "Billing Team"

Event: { "templateName": "invoice", "templateData": {...} }
Result: "Billing Team" <billing@company.com>
```

**Scenario 2: Per-email override (support ticket reply)**
```
Template: support-reply/metadata.json
  "senderEmail": "support@company.com"
  "senderName": "Support Team"

Event: {
  "templateName": "support-reply",
  "senderOverride": {
    "name": "Sarah from Support",
    "email": "sarah@company.com"
  }
}
Result: "Sarah from Support" <sarah@company.com>
```

**Scenario 3: Dynamic sender via templateData**
```
Template: welcome/metadata.json
  "senderEmail": "{{ senderEmail | default('noreply@company.com') }}"

Event: { "templateData": { "senderEmail": "sales@company.com" } }
Result: Uses sales@company.com

Event: { "templateData": {} }  // No senderEmail
Result: Uses noreply@company.com (default)
```

### When to Use Each

| Use Case | Solution |
|----------|----------|
| All emails from same address | Set `DefaultSenderEmail` in CloudFormation |
| Different sender per template type | Set `senderEmail` in template metadata |
| Dynamic sender (multi-tenant, per-agent) | Use `senderOverride` in event |
| Flexible default with override option | Use `{{ senderEmail \| default('...') }}` in template |

---

## Template System

### Template Structure

Each template is a folder in S3:
```
templates/
└── welcome/
    ├── template.html      # HTML version
    ├── template.txt       # Plain text version
    └── metadata.json      # Configuration
```

### metadata.json Fields

```json
{
  "subject": "Welcome to {{ companyName }}!",
  "senderName": "{{ companyName }} Team",
  "senderEmail": "{{ senderEmail | default('noreply@company.com') }}",
  "description": "Welcome email for new users",
  "requiredVariables": ["userName", "companyName"],
  "optionalVariables": ["logoUrl", "ctaUrl", "ctaText", "helpUrl", "senderEmail"],
  "version": "2.0.0"
}
```

### Jinja2 Variables

| Syntax | Description |
|--------|-------------|
| `{{ variable }}` | Output variable value |
| `{{ var \| default('fallback') }}` | Use fallback if var is empty/missing |
| `{% if condition %}...{% endif %}` | Conditional content |
| `{% for item in list %}...{% endfor %}` | Loop over list |

### Template Versioning

Templates use semantic versioning for safe updates:

| Scenario | Seeder Action |
|----------|---------------|
| Template doesn't exist | **Installed** |
| Starter version > your version | **Updated** |
| Starter version = your version | **Skipped** |
| Starter version < your version | **Skipped** (your customizations preserved) |

**To preserve customizations:** Bump your template's version number after editing.

---

## Email Tracking

### Status Values

| Status | Meaning | Final? |
|--------|---------|--------|
| `sent` | Accepted by SES, awaiting feedback | No |
| `delivered` | SES confirmed delivery to recipient's server | Yes |
| `opened` | Recipient opened email (tracking pixel) | Yes |
| `soft_bounced` | Temporary failure, will retry | No |
| `bounced` | Permanent failure (address doesn't exist) | Yes |
| `complained` | Recipient marked as spam | Yes |
| `failed` | Processing error (template, suppression, etc.) | Yes |

### Status Priority

Higher statuses can't be overwritten by lower ones:
```
opened > delivered > bounced > complained > soft_bounced > failed > sent
```

This prevents race conditions (e.g., "delivered" arriving after "opened").

### Tracking Record Fields

| Field | Description |
|-------|-------------|
| `emailId` | Unique identifier (auto-generated or custom) |
| `toEmail` | Recipient email address |
| `status` | Current delivery status |
| `sesMessageId` | SES message ID for correlation |
| `timestamp` | When email was sent |
| `deliveredAt` | When delivery was confirmed |
| `openedAt` | When first opened |
| `openCount` | Number of times opened |
| `errorMessage` | Error details if failed |
| `metadata` | Custom metadata from event |

---

## Checking Email Status

### By Email ID
```bash
aws dynamodb get-item \
  --table-name {STACK}-EmailTracking \
  --key '{"emailId": {"S": "email-123"}}'
```

### By Recipient (recent emails)
```bash
aws dynamodb query \
  --table-name {STACK}-EmailTracking \
  --index-name to-email-timestamp-index \
  --key-condition-expression "toEmail = :email" \
  --expression-attribute-values '{":email": {"S": "user@example.com"}}' \
  --scan-index-forward false \
  --limit 10
```

### Check if Email Was Opened
```bash
aws dynamodb get-item \
  --table-name {STACK}-EmailTracking \
  --key '{"emailId": {"S": "email-123"}}' \
  --projection-expression "#s, openedAt, openCount" \
  --expression-attribute-names '{"#s": "status"}'
```

**Response interpretation:**
- `status: "opened"` → Email was opened
- `openedAt` → Timestamp of first open
- `openCount` → Total number of opens

---

## Suppression List

### What Gets Suppressed

| Event | Action |
|-------|--------|
| Hard bounce (address doesn't exist) | Immediately suppressed |
| Spam complaint | Immediately suppressed |
| 15+ soft bounces in 30 days | Promoted to permanent suppression |

**Note:** A single failed email chain (retry exhausted) does NOT suppress the address. The email is marked as "failed" and you can resend if appropriate.

### Check if Email is Suppressed
```bash
aws dynamodb get-item \
  --table-name {STACK}-Suppression \
  --key '{"email": {"S": "user@example.com"}}'
```

**If item exists:** Email is blocked. Check `reason`:
- `hard-bounce` - Address doesn't exist
- `soft-bounce-exceeded` - 15+ soft bounces in 30 days
- `spam-complaint` - User reported spam

**Note:** If an email has `failed` status (retry exhausted), the address is NOT suppressed. You can resend if appropriate.

### Remove from Suppression (Use with Caution!)
```bash
aws dynamodb delete-item \
  --table-name {STACK}-Suppression \
  --key '{"email": {"S": "user@example.com"}}'
```

**Warning:** Removing hard bounces damages SES reputation. Only remove if you're certain the address is now valid.

---

## Bounce Rate Protection

### How It Works

1. System tracks daily bounce rate: `bounces / emails_sent * 100`
2. If rate exceeds threshold (default 5%), new emails are blocked
3. Rate resets daily at midnight UTC
4. Protects SES reputation (AWS suspends at ~10%)

### Check Current Bounce Rate
```bash
# Count today's emails
TODAY=$(date -u +%Y-%m-%d)
aws dynamodb query \
  --table-name {STACK}-EmailTracking \
  --index-name date-partition-index \
  --key-condition-expression "datePartition = :date" \
  --expression-attribute-values "{\":date\": {\"S\": \"$TODAY\"}}" \
  --select COUNT

# Count today's bounces
aws dynamodb query \
  --table-name {STACK}-Suppression \
  --index-name suppression-type-date-index \
  --key-condition-expression "suppressionType = :type AND begins_with(addedToSuppressionDate, :date)" \
  --expression-attribute-values "{\":type\": {\"S\": \"bounce\"}, \":date\": {\"S\": \"$TODAY\"}}" \
  --select COUNT

# Calculate: bounce_count / email_count * 100
```

### If Bounce Rate Exceeded

Options:
1. **Wait** - Rate resets at midnight UTC
2. **Clean email list** - Remove invalid addresses before sending
3. **Increase threshold** - Update CloudFormation parameter (not recommended)

---

## Soft Bounce Retry System

### How It Works

1. Soft bounce received (MailboxFull, etc.)
2. Email queued for single retry (15-minute delay)
3. Retry attempt made
4. If retry succeeds → delivered
5. If retry fails → marked as "failed" (NOT suppressed)
6. Customer can resend if appropriate

### Cross-Campaign Protection

If an email address accumulates 15+ soft bounces within a 30-day rolling window (across all campaigns), it is permanently suppressed. This protects against truly problematic addresses while avoiding aggressive suppression from a single failed email chain.

### Retry Timeline
```
T+0:00  - Original send, soft bounce
T+0:15  - Retry attempt
T+0:15+ - If retry fails → status "failed" (you can resend)
```

### Understanding "failed" vs "suppressed"

| Status | Meaning | Can Resend? |
|--------|---------|-------------|
| `failed` | Retry exhausted, but address NOT suppressed | Yes - your choice |
| `bounced` | Hard bounce, address permanently suppressed | No - will be blocked |
| `soft_bounced` | Temporary failure, retry scheduled | Wait for retry result |

### Check Retry Status
```bash
aws dynamodb query \
  --table-name {STACK}-EmailTracking \
  --index-name to-email-timestamp-index \
  --key-condition-expression "toEmail = :email" \
  --filter-expression "originalEmailId = :orig" \
  --expression-attribute-values '{":email": {"S": "user@example.com"}, ":orig": {"S": "email-123"}}'
```

---

## Starter Templates

### Available Templates

| Template | Description | Required Variables |
|----------|-------------|-------------------|
| `welcome` | New user registration | `userName`, `companyName` |
| `password-reset` | Password reset with expiration | `userName`, `companyName`, `resetLink` |
| `purchase-confirmation` | Order/purchase confirmation | `customerName`, `orderId`, `productName`, `price`, `companyName` |
| `email-verification` | Email address verification | `userName`, `verificationUrl`, `companyName` |
| `invoice` | Professional invoice with line items | `customerName`, `invoiceNumber`, `invoiceDate`, `items`, `total`, `companyName` |
| `subscription-renewal` | Subscription renewal reminder | `userName`, `planName`, `renewalDate`, `amount`, `companyName` |
| `contact-form-notification` | Internal contact form notification | `formSenderName`, `formSenderEmail`, `message` |
| `shipping-notification` | Order shipped with tracking | `customerName`, `orderId`, `trackingNumber`, `carrier`, `companyName` |

All templates feature:
- Responsive design (mobile breakpoint at 620px)
- Optional `logoUrl` for branding
- Plain text fallback versions
- Outlook VML button compatibility

### Install Starter Templates
```bash
aws lambda invoke \
  --function-name {STACK}-TemplateSeeder \
  --output text /dev/stdout
```

### Response Format
```json
{
  "statusCode": 200,
  "installed": ["welcome"],
  "updated": ["password-reset"],
  "skipped": [],
  "message": "Installed 1, updated 1, skipped 0"
}
```

---

## CloudWatch Alarms

### Available Alarms

| Alarm | Trigger | Severity |
|-------|---------|----------|
| `EventBridgeDLQDepthAlarm` | Failed Lambda invocations | Medium |
| `RetryDLQDepthAlarm` | Failed retry attempts | Medium |
| `FeedbackDLQDepthAlarm` | Failed feedback processing | Medium |
| `EmailSenderErrorsAlarm` | Lambda errors | Medium |
| `FeedbackProcessorErrorsAlarm` | Lambda errors | Medium |
| `SESBounceRateAlarm` | Bounce rate > 3% | **Critical** |
| `SESComplaintRateAlarm` | Complaint rate > 0.05% | **Critical** |

### Check Alarm Status
```bash
aws cloudwatch describe-alarms \
  --alarm-name-prefix {STACK} \
  --query 'MetricAlarms[].{Name:AlarmName,State:StateValue}'
```

---

## Common Questions

### "How do I send from a different email address?"

Three options:
1. **Per-email:** Use `senderOverride.email` in the event
2. **Per-template:** Set `senderEmail` in template metadata
3. **Global default:** Update `DefaultSenderEmail` CloudFormation parameter

### "How do I check if my email was delivered?"

Query the tracking table:
```bash
aws dynamodb get-item \
  --table-name {STACK}-EmailTracking \
  --key '{"emailId": {"S": "your-email-id"}}'
```
Check `status` field: `delivered` = success, `bounced` = failed.

### "How do I check if my email was opened?"

Same query as above. Check:
- `status: "opened"` = yes, it was opened
- `openedAt` = when it was first opened
- `openCount` = how many times

### "Why is my email not being sent?"

Check in order:
1. Is recipient suppressed? (Query suppression table)
2. Is bounce rate exceeded? (Check daily bounce rate)
3. Is template valid? (Check S3 for all required files)
4. Is sender verified in SES? (Check SES console)

### "How do I remove someone from the suppression list?"

```bash
aws dynamodb delete-item \
  --table-name {STACK}-Suppression \
  --key '{"email": {"S": "user@example.com"}}'
```
**Warning:** Only do this if you're certain the address is now valid.

### "How do I customize a starter template?"

1. Download from S3: `aws s3 cp s3://{BUCKET}/templates/welcome/ ./welcome/ --recursive`
2. Edit the files
3. Bump version in metadata.json (e.g., `1.0.0` → `1.1.0`)
4. Upload back: `aws s3 cp ./welcome/ s3://{BUCKET}/templates/welcome/ --recursive`

### "How do I set a default sender for a specific template?"

Edit the template's `metadata.json`:
```json
{
  "senderEmail": "billing@company.com",
  "senderName": "Billing Team"
}
```

### "How do I make the sender dynamic but with a fallback?"

Use Jinja2 default filter in `metadata.json`:
```json
{
  "senderEmail": "{{ senderEmail | default('noreply@company.com') }}"
}
```
Then pass `senderEmail` in `templateData` when you want to override.

### "How do I track emails by campaign?"

Include metadata in your event:
```json
{
  "templateData": {...},
  "metadata": {
    "campaignId": "summer-sale-2024"
  }
}
```
Query by scanning with filter on `metadata.campaignId`.

### "How do I get notified of bounces/complaints?"

CloudWatch alarms notify the `AdminEmail` address. For programmatic access:
1. Subscribe to the SNS topic (from stack outputs)
2. Or query the suppression table periodically
