# Integration Guide

This guide shows how to send emails from your application using SESEmailEngine.

## Overview

SESEmailEngine uses EventBridge for async email requests. Your application publishes events to the custom event bus, and the system handles template rendering, sending, tracking, and bounce management.

```
Your App → EventBridge → SESEmailEngine → SES → Recipient
```

## Prerequisites

Before integrating, ensure you have:
1. **Deployed SESMailEngine** using `python install.py` (see [SETUP.md](SETUP.md))
2. **Noted your stack name** (default: `sesmailengine`)
3. **IAM permissions** to publish to EventBridge (see IAM section below)

## Quick Start

### 1. Get Your Event Bus Name

After deployment, the installer displays your EventBridge bus name. You can also get it from CloudFormation outputs:

```bash
aws cloudformation describe-stacks \
  --stack-name sesmailengine \
  --query 'Stacks[0].Outputs[?OutputKey==`EventBusName`].OutputValue' \
  --output text
```

### 2. Send Your First Email

```python
import boto3
import json

events = boto3.client('events')

response = events.put_events(
    Entries=[{
        'Source': 'my.application',
        'DetailType': 'Email Request',
        'EventBusName': 'sesmailengine-EmailBus',  # Use your stack name
        'Detail': json.dumps({
            'to': 'recipient@example.com',
            'templateName': 'welcome',
            'templateData': {
                'userName': 'John',
                'companyName': 'My Company'
            }
        })
    }]
)

print(f"Sent: {response}")
```

---

## Event Schema

### Complete Event Format

```json
{
  "source": "my.application",
  "detail-type": "Email Request",
  "detail": {
    "emailId": "optional-custom-id-123",
    "to": "recipient@example.com",
    "subject": "Optional subject override",
    "templateName": "welcome",
    "templateData": {
      "userName": "John Doe",
      "companyName": "Acme Corp",
      "ctaUrl": "https://example.com/dashboard"
    },
    "senderOverride": {
      "email": "custom@yourcompany.com",
      "name": "Custom Sender Name"
    },
    "replyTo": "support@yourcompany.com",
    "metadata": {
      "campaignId": "welcome-2024",
      "userId": "user-456",
      "source": "signup-form"
    }
  }
}
```

### Field Reference

| Field | Required | Description |
|-------|----------|-------------|
| `to` | Yes | Recipient email address |
| `templateName` | Yes | Name of template folder in S3 |
| `templateData` | No | Variables to render in template |
| `emailId` | No | Custom ID for tracking (auto-generated if not provided) |
| `subject` | No | Overrides template subject |
| `senderOverride` | No | Overrides sender email and name |
| `replyTo` | No | Reply-To address (where replies go when recipient clicks "Reply") |
| `metadata` | No | Custom data stored with tracking record |

### Sender Override

Override the default sender for specific emails:

```json
{
  "senderOverride": {
    "email": "sales@yourcompany.com",
    "name": "Sales Team"
  }
}
```

**Sender Priority (highest to lowest):**
1. `senderOverride` in event
2. `senderEmail` in template metadata.json
3. `DefaultSenderEmail` CloudFormation parameter

### Reply-To Address

Specify where replies should go when recipients click "Reply":

```json
{
  "replyTo": "support@yourcompany.com"
}
```

**Use cases:**
- Send from `noreply@company.com` but have replies go to `support@company.com`
- Contact form notifications: replies go directly to the form submitter
- Sales emails: replies go to the sales rep, not the generic sender

**Reply-To Priority (highest to lowest):**
1. `replyTo` in EventBridge event (explicit per-email override)
2. `replyTo` in template metadata.json (template default, supports Jinja2 variables)

**Example with template metadata:**

The `contact-form-notification` template uses `"replyTo": "{{ formSenderEmail }}"` in metadata.json, so replies automatically go to the person who submitted the contact form.

**Note:** Reply-To is different from Return-Path:
- **Reply-To**: Where user replies go (cosmetic, user-facing)
- **Return-Path**: Where bounces go (SMTP envelope, configured at SES account level)

### Metadata

Store custom data with each email for your own tracking:

```json
{
  "metadata": {
    "campaignId": "black-friday-2024",
    "userId": "user-123",
    "orderId": "order-456",
    "source": "checkout-flow"
  }
}
```

Metadata is stored in DynamoDB and can be queried later.

---

## Code Examples

### Python

```python
import boto3
import json
from typing import Optional

class EmailClient:
    def __init__(self, event_bus_name: str):
        self.events = boto3.client('events')
        self.event_bus_name = event_bus_name
    
    def send_email(
        self,
        to: str,
        template_name: str,
        template_data: dict,
        subject: Optional[str] = None,
        sender_email: Optional[str] = None,
        sender_name: Optional[str] = None,
        reply_to: Optional[str] = None,
        metadata: Optional[dict] = None,
        email_id: Optional[str] = None,
    ) -> dict:
        """Send an email via SESEmailEngine."""
        
        detail = {
            'to': to,
            'templateName': template_name,
            'templateData': template_data,
        }
        
        if email_id:
            detail['emailId'] = email_id
        if subject:
            detail['subject'] = subject
        if sender_email:
            detail['senderOverride'] = {
                'email': sender_email,
                'name': sender_name,
            }
        if reply_to:
            detail['replyTo'] = reply_to
        if metadata:
            detail['metadata'] = metadata
        
        response = self.events.put_events(
            Entries=[{
                'Source': 'my.application',
                'DetailType': 'Email Request',
                'EventBusName': self.event_bus_name,
                'Detail': json.dumps(detail),
            }]
        )
        
        return response

# Usage
client = EmailClient('my-email-engine-EmailBus')

# Simple email
client.send_email(
    to='user@example.com',
    template_name='welcome',
    template_data={'userName': 'John'},
)

# Email with all options
client.send_email(
    to='user@example.com',
    template_name='order-confirmation',
    template_data={
        'userName': 'John',
        'orderId': 'ORD-123',
        'total': '$99.99',
    },
    subject='Your Order #ORD-123',
    sender_email='orders@mystore.com',
    sender_name='MyStore Orders',
    metadata={
        'orderId': 'ORD-123',
        'userId': 'user-456',
    },
)
```

### Node.js / TypeScript

```typescript
import { EventBridgeClient, PutEventsCommand } from '@aws-sdk/client-eventbridge';

interface SendEmailOptions {
  to: string;
  templateName: string;
  templateData: Record<string, any>;
  subject?: string;
  senderOverride?: {
    email: string;
    name?: string;
  };
  replyTo?: string;
  metadata?: Record<string, any>;
  emailId?: string;
}

class EmailClient {
  private client: EventBridgeClient;
  private eventBusName: string;

  constructor(eventBusName: string) {
    this.client = new EventBridgeClient({});
    this.eventBusName = eventBusName;
  }

  async sendEmail(options: SendEmailOptions): Promise<void> {
    const detail: Record<string, any> = {
      to: options.to,
      templateName: options.templateName,
      templateData: options.templateData,
    };

    if (options.emailId) detail.emailId = options.emailId;
    if (options.subject) detail.subject = options.subject;
    if (options.senderOverride) detail.senderOverride = options.senderOverride;
    if (options.replyTo) detail.replyTo = options.replyTo;
    if (options.metadata) detail.metadata = options.metadata;

    const command = new PutEventsCommand({
      Entries: [{
        Source: 'my.application',
        DetailType: 'Email Request',
        EventBusName: this.eventBusName,
        Detail: JSON.stringify(detail),
      }],
    });

    await this.client.send(command);
  }
}

// Usage
const emailClient = new EmailClient('my-email-engine-EmailBus');

// Simple email
await emailClient.sendEmail({
  to: 'user@example.com',
  templateName: 'welcome',
  templateData: { userName: 'John' },
});

// Email with options
await emailClient.sendEmail({
  to: 'user@example.com',
  templateName: 'password-reset',
  templateData: {
    userName: 'John',
    resetLink: 'https://example.com/reset/abc123',
  },
  senderOverride: {
    email: 'security@myapp.com',
    name: 'MyApp Security',
  },
  metadata: {
    userId: 'user-456',
    requestIp: '192.168.1.1',
  },
});
```

### Java

```java
import software.amazon.awssdk.services.eventbridge.EventBridgeClient;
import software.amazon.awssdk.services.eventbridge.model.PutEventsRequest;
import software.amazon.awssdk.services.eventbridge.model.PutEventsRequestEntry;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.Map;
import java.util.HashMap;

public class EmailClient {
    private final EventBridgeClient client;
    private final String eventBusName;
    private final ObjectMapper mapper;

    public EmailClient(String eventBusName) {
        this.client = EventBridgeClient.create();
        this.eventBusName = eventBusName;
        this.mapper = new ObjectMapper();
    }

    public void sendEmail(
        String to,
        String templateName,
        Map<String, Object> templateData
    ) throws Exception {
        sendEmail(to, templateName, templateData, null, null, null, null);
    }

    public void sendEmail(
        String to,
        String templateName,
        Map<String, Object> templateData,
        String subject,
        Map<String, String> senderOverride,
        String replyTo,
        Map<String, Object> metadata
    ) throws Exception {
        Map<String, Object> detail = new HashMap<>();
        detail.put("to", to);
        detail.put("templateName", templateName);
        detail.put("templateData", templateData);

        if (subject != null) detail.put("subject", subject);
        if (senderOverride != null) detail.put("senderOverride", senderOverride);
        if (replyTo != null) detail.put("replyTo", replyTo);
        if (metadata != null) detail.put("metadata", metadata);

        PutEventsRequest request = PutEventsRequest.builder()
            .entries(PutEventsRequestEntry.builder()
                .source("my.application")
                .detailType("Email Request")
                .eventBusName(eventBusName)
                .detail(mapper.writeValueAsString(detail))
                .build())
            .build();

        client.putEvents(request);
    }
}

// Usage
EmailClient emailClient = new EmailClient("my-email-engine-EmailBus");

// Simple email
emailClient.sendEmail(
    "user@example.com",
    "welcome",
    Map.of("userName", "John")
);

// Email with options
emailClient.sendEmail(
    "user@example.com",
    "invoice",
    Map.of(
        "userName", "John",
        "invoiceId", "INV-123",
        "amount", "$500.00"
    ),
    "Invoice #INV-123",
    Map.of("email", "billing@mycompany.com", "name", "Billing"),
    "billing-support@mycompany.com",
    Map.of("invoiceId", "INV-123", "customerId", "cust-456")
);
```

---

## Batch Sending

Send multiple emails in a single API call (up to 10 per request):

```python
import boto3
import json

events = boto3.client('events')

users = [
    {'email': 'user1@example.com', 'name': 'Alice'},
    {'email': 'user2@example.com', 'name': 'Bob'},
    {'email': 'user3@example.com', 'name': 'Charlie'},
]

entries = [
    {
        'Source': 'my.application',
        'DetailType': 'Email Request',
        'EventBusName': 'my-email-engine-EmailBus',
        'Detail': json.dumps({
            'to': user['email'],
            'templateName': 'newsletter',
            'templateData': {'userName': user['name']},
            'metadata': {'campaign': 'december-newsletter'},
        }),
    }
    for user in users
]

# EventBridge allows max 10 entries per request
for i in range(0, len(entries), 10):
    batch = entries[i:i+10]
    response = events.put_events(Entries=batch)
    
    # Check for failures
    if response['FailedEntryCount'] > 0:
        for j, entry in enumerate(response['Entries']):
            if 'ErrorCode' in entry:
                print(f"Failed: {users[i+j]['email']} - {entry['ErrorMessage']}")
```

---

## IAM Permissions

Your application (Lambda, EC2, ECS, etc.) needs permission to publish events to the SESEmailEngine EventBridge bus. Without this permission, your service will get `AccessDeniedException` errors.

### Option 1: Get Policy from CloudFormation Output

After deploying SESEmailEngine, the stack outputs a ready-to-use IAM policy:

**Via AWS CLI:**
```bash
aws cloudformation describe-stacks \
  --stack-name my-email-engine \
  --query 'Stacks[0].Outputs[?OutputKey==`CustomerIAMPolicy`].OutputValue' \
  --output text
```

**Via AWS Console:**
1. Go to CloudFormation → Stacks → your-stack-name
2. Click the "Outputs" tab
3. Find `CustomerIAMPolicy` and copy the JSON value

### Option 2: Create Policy Manually

If you prefer to create the policy yourself, use this template:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSESEmailEnginePutEvents",
      "Effect": "Allow",
      "Action": "events:PutEvents",
      "Resource": "arn:aws:events:REGION:ACCOUNT_ID:event-bus/STACK_NAME-EmailBus"
    }
  ]
}
```

Replace:
- `REGION` - Your AWS region (e.g., `us-east-1`)
- `ACCOUNT_ID` - Your 12-digit AWS account ID
- `STACK_NAME` - The CloudFormation stack name you used

**Example (filled in):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSESEmailEnginePutEvents",
      "Effect": "Allow",
      "Action": "events:PutEvents",
      "Resource": "arn:aws:events:us-east-1:123456789012:event-bus/my-email-engine-EmailBus"
    }
  ]
}
```

### Attaching the Policy via AWS Console

**For Lambda Functions:**
1. Go to Lambda → Functions → your-function
2. Click "Configuration" → "Permissions"
3. Click the execution role name (opens IAM)
4. Click "Add permissions" → "Create inline policy"
5. Click "JSON" tab and paste the policy above
6. Click "Review policy" → name it `SESEmailEngineAccess` → "Create policy"

**For EC2 Instances:**
1. Go to IAM → Roles → find your EC2 instance role
2. Click "Add permissions" → "Create inline policy"
3. Click "JSON" tab and paste the policy above
4. Click "Review policy" → name it `SESEmailEngineAccess` → "Create policy"

**For ECS Tasks:**
1. Go to IAM → Roles → find your ECS task role
2. Click "Add permissions" → "Create inline policy"
3. Click "JSON" tab and paste the policy above
4. Click "Review policy" → name it `SESEmailEngineAccess` → "Create policy"

### Attaching the Policy via AWS CLI

```bash
# Get the EventBus ARN
EVENT_BUS_ARN=$(aws cloudformation describe-stacks \
  --stack-name my-email-engine \
  --query 'Stacks[0].Outputs[?OutputKey==`EventBusArn`].OutputValue' \
  --output text)

# Create the policy document
cat > /tmp/ses-email-engine-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSESEmailEnginePutEvents",
      "Effect": "Allow",
      "Action": "events:PutEvents",
      "Resource": "${EVENT_BUS_ARN}"
    }
  ]
}
EOF

# Attach to your role (replace YOUR_ROLE_NAME)
aws iam put-role-policy \
  --role-name YOUR_ROLE_NAME \
  --policy-name SESEmailEngineAccess \
  --policy-document file:///tmp/ses-email-engine-policy.json
```

### Verify Permissions

Test that your service can publish events:

```python
import boto3

events = boto3.client('events')

try:
    response = events.put_events(
        Entries=[{
            'Source': 'test.permissions',
            'DetailType': 'Email Request',
            'EventBusName': 'my-email-engine-EmailBus',
            'Detail': '{"to":"test@example.com","templateName":"test","templateData":{}}'
        }]
    )
    if response['FailedEntryCount'] == 0:
        print("✅ Permissions configured correctly!")
    else:
        print(f"❌ Failed: {response['Entries'][0].get('ErrorMessage')}")
except Exception as e:
    print(f"❌ Error: {e}")
```

### Common Permission Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `AccessDeniedException` | Missing IAM policy | Attach the policy above to your service's role |
| `ResourceNotFoundException` | Wrong event bus name | Check the `EventBusName` output from CloudFormation |
| `InvalidParameterException` | Wrong event bus ARN in policy | Verify the ARN matches your stack's EventBus |

---

## Tracking Emails

### Query by Email ID

If you provided a custom `emailId`, query the tracking table:

```python
import boto3

dynamodb = boto3.client('dynamodb')

response = dynamodb.get_item(
    TableName='my-email-engine-EmailTracking',
    Key={'emailId': {'S': 'my-custom-id-123'}},
)

if 'Item' in response:
    item = response['Item']
    print(f"Status: {item['status']['S']}")
    print(f"Sent at: {item['timestamp']['S']}")
```

### Query by Recipient

Find all emails sent to a specific address:

```python
response = dynamodb.query(
    TableName='my-email-engine-EmailTracking',
    IndexName='to-email-timestamp-index',
    KeyConditionExpression='toEmail = :email',
    ExpressionAttributeValues={':email': {'S': 'user@example.com'}},
)

for item in response['Items']:
    print(f"{item['emailId']['S']}: {item['status']['S']}")
```

### Tracking Retry Attempts

When a soft bounce occurs, the system automatically retries the email. Each retry creates a NEW tracking record with:
- A new `emailId` (auto-generated)
- `originalEmailId` pointing to the first email attempt
- `retryAttempt` counter (0 for original, 1 for retry)

**Why new records per retry?**
- Full audit trail - every attempt is recorded
- Simple INSERT operations with idempotency protection
- No risk of overwriting data from race conditions

**Query all attempts for a specific email using `original-email-id-index` GSI:**

```python
import boto3

dynamodb = boto3.client('dynamodb')

# Find all attempts (original + retries) for a specific email
# Use the originalEmailId from your first send
response = dynamodb.query(
    TableName='my-email-engine-EmailTracking',
    IndexName='original-email-id-index',
    KeyConditionExpression='originalEmailId = :origId',
    ExpressionAttributeValues={':origId': {'S': 'email-abc123'}},
    ScanIndexForward=True,  # Oldest first
)

for item in response['Items']:
    email_id = item['emailId']['S']
    retry = int(item.get('retryAttempt', {}).get('N', '0'))
    status = item['status']['S']
    
    if retry > 0:
        print(f"  └─ Retry #{retry}: {email_id} → {status}")
    else:
        print(f"Original: {email_id} → {status}")
```

**Example output:**
```
Original: email-abc123 → soft_bounced
  └─ Retry #1: email-def456 → delivered
```

**Understanding the retry chain:**
- Original email (`retryAttempt=0`) shows `soft_bounced` status
- Retry email (`retryAttempt=1`) shows final outcome (`delivered` or `failed`)
- Both records have the same `originalEmailId` value
- Query by `originalEmailId` returns ONLY the retry chain for that specific email (not all emails to the recipient)

**Get final outcome for an email:**

```python
# Query all attempts and get the latest one
response = dynamodb.query(
    TableName='my-email-engine-EmailTracking',
    IndexName='original-email-id-index',
    KeyConditionExpression='originalEmailId = :origId',
    ExpressionAttributeValues={':origId': {'S': 'email-abc123'}},
    ScanIndexForward=False,  # Newest first
    Limit=1,  # Get only the latest attempt
)

if response['Items']:
    final = response['Items'][0]
    print(f"Final status: {final['status']['S']}")
```

### Email Statuses

Understanding email statuses helps you track delivery success and troubleshoot issues.

| Status | Meaning | What to Do |
|--------|---------|------------|
| `sent` | Email accepted by SES, awaiting delivery confirmation | Wait for delivery/bounce feedback |
| `delivered` | SES confirmed delivery to recipient's mail server | Success! Email reached inbox |
| `opened` | Recipient opened the email (tracking pixel loaded) | Engagement confirmed |
| `soft_bounced` | Temporary failure (mailbox full, etc.) | System will retry automatically (once) |
| `bounced` | Permanent failure (address doesn't exist) | Remove from your mailing list |
| `complained` | Recipient marked email as spam | Never email this address again |
| `failed` | Processing error or retry exhausted | Check `errorMessage` field - you can resend if appropriate |

**Understanding "failed" Status:**

The `failed` status indicates the email was not sent, but the recipient is NOT suppressed. Common reasons include:
- **Template errors**: Template not found or rendering failed
- **Validation errors**: Invalid email format or missing required fields
- **Bounce rate exceeded**: Daily bounce rate threshold exceeded (temporary)
- **Retry exhausted**: Soft bounce retry failed (you can resend later)
- **Suppressed recipient**: Recipient is on suppression list (check suppression table)

When you see `failed` status, check the `errorMessage` field for details. Unlike `bounced` or `complained`, a `failed` email does NOT automatically suppress the recipient - you retain control over whether to retry.

**Status Lifecycle:**

```
┌──────┐     ┌───────────┐     ┌──────────┐
│ sent │────▶│ delivered │────▶│  opened  │
└──────┘     └───────────┘     └──────────┘
    │
    │        ┌─────────────┐     ┌─────────┐
    ├───────▶│ soft_bounced│────▶│ failed  │  (retry exhausted - you can resend)
    │        └─────────────┘     └─────────┘
    │              │
    │              └───────────▶ delivered  (retry succeeded)
    │
    │        ┌─────────────┐
    └───────▶│   bounced   │  (permanent - address suppressed)
             └─────────────┘
```

**Terminal States:** `bounced` and `complained` are final - the recipient is added to the suppression list and will never receive emails again (unless manually removed).

**Status Priority:** `opened` > `delivered` > `bounced` > `complained` > `soft_bounced` > `failed` > `sent`

Higher priority statuses cannot be overwritten by lower priority ones. This prevents race conditions when SES events arrive out of order (e.g., "open" event arrives before "delivery" event).

### Soft Bounce Retry Behavior

When a soft bounce occurs (e.g., mailbox full), the system automatically retries once:

| Attempt | Delay | Action |
|---------|-------|--------|
| 1st retry | 15 minutes | Retry via SQS |
| After retry fails | - | Mark as "failed" (customer can resend) |

**Why single retry?** This provides faster feedback to customers while still handling temporary issues. If the retry fails, the email is marked as "failed" (not suppressed), allowing customers to decide whether to resend.

**Cross-Campaign Protection:** If an email address accumulates 15 soft bounces within a 30-day window (across all campaigns), it is permanently suppressed. This protects against truly problematic addresses while avoiding aggressive suppression from a single failed email chain.

**Soft bounce types that trigger retry:**
- `MailboxFull` - Recipient's mailbox is full
- `MessageTooLarge` - Email too large for recipient
- `ContentRejected` - Spam filter rejection
- `General` - Unspecified temporary failure

**Hard bounces (immediate suppression, no retry):**
- `NoEmail` - Address doesn't exist
- `Suppressed` - Already on SES suppression list

### Email Processing Flow

This diagram shows every path through the Email Sender Lambda and when tracking records are created:

```
Event Arrives (EventBridge or SQS)
    │
    ├─► Parse & Validate Event
    │       │
    │       └─► Invalid email / missing fields
    │               └─► ✅ Track as "failed" → Event consumed
    │
    ├─► Check Suppression List
    │       │
    │       └─► Recipient is suppressed
    │               └─► ✅ Track as "failed" → Event consumed
    │
    ├─► Check Bounce Rate
    │       │
    │       ├─► Exceeded + EventBridge source
    │       │       └─► ✅ Track as "failed" → Event consumed
    │       │
    │       └─► Exceeded + SQS source
    │               └─► ❌ No tracking → Message stays in queue
    │
    ├─► Load & Render Template
    │       │
    │       └─► Template not found / render error
    │               └─► ✅ Track as "failed" → Event consumed
    │
    ├─► Send via SES
    │       │
    │       ├─► Success
    │       │       └─► ✅ Track as "sent" → Event consumed
    │       │
    │       ├─► Permanent error (invalid address, etc.)
    │       │       └─► ✅ Track as "failed" → Event consumed
    │       │
    │       └─► Retryable error (throttling, rate limit)
    │               └─► ❌ No tracking → EventBridge retries (3x)
    │                       │
    │                       └─► All retries exhausted
    │                               └─► ❌ No tracking → DLQ
    │
    └─► AWS Infrastructure Error
            └─► ❌ No tracking → Retries, then DLQ
```

**Important: SES Throttling Behavior**

When SES returns a throttling error (rate limit exceeded), the system does NOT create a tracking record. Instead:
1. Lambda throws an exception
2. EventBridge retries the event (3 attempts with exponential backoff)
3. If all retries fail, the event goes to the EventBridge DLQ

This means throttled emails that exhaust all retries will have NO tracking record in DynamoDB. To find these emails, check the EventBridge DLQ. See [TROUBLESHOOTING.md](TROUBLESHOOTING.md#throttling--toomanyrequestsexception) for details.

**Key Principle: No Silent Email Loss**

| Scenario | Tracked in DynamoDB? | Event Consumed? |
|----------|---------------------|-----------------|
| Email sent successfully | ✅ Yes (`sent`) | Yes |
| Recipient suppressed | ✅ Yes (`failed`) | Yes |
| Recipient suppressed (SQS retry) | ✅ Yes (`failed`) | Yes |
| Bounce rate exceeded (EventBridge) | ✅ Yes (`failed`) | Yes |
| Bounce rate exceeded (SQS retry) | ✅ Yes (`failed`) | Yes |
| Template error | ✅ Yes (`failed`) | Yes |
| Validation error | ✅ Yes (`failed`) | Yes |
| SES permanent error | ✅ Yes (`failed`) | Yes |
| SES retryable error | ❌ No | No - retries |
| AWS transient error | ❌ No | No - retries |
| Unknown error | ❌ No | No - goes to DLQ |

Every consumed event creates a tracking record so you can query what happened. Retryable errors don't create records because the event will be processed again.

---

## Best Practices

### 1. Use Meaningful Email IDs

Provide your own `emailId` for easier tracking:

```python
{
    'emailId': f'order-confirmation-{order_id}',
    # ...
}
```

### 2. Include Metadata for Analytics

Store data you'll need later:

```python
{
    'metadata': {
        'campaignId': 'black-friday-2024',
        'segment': 'premium-users',
        'abTest': 'variant-b',
    }
}
```

### 3. Handle EventBridge Failures

Check the response for failed entries:

```python
response = events.put_events(Entries=entries)

if response['FailedEntryCount'] > 0:
    for entry in response['Entries']:
        if 'ErrorCode' in entry:
            # Log or retry
            print(f"Failed: {entry['ErrorCode']} - {entry['ErrorMessage']}")
```

### 4. Don't Send to Suppressed Addresses

The system automatically blocks suppressed emails, but you can check proactively:

```python
response = dynamodb.get_item(
    TableName='my-email-engine-Suppression',
    Key={'email': {'S': email.lower()}},
)

if 'Item' in response:
    print(f"Email is suppressed: {response['Item']['reason']['S']}")
```

### 5. Rate Limit Your Sends

If sending large batches, add delays to avoid overwhelming SES:

```python
import time

for batch in batches:
    events.put_events(Entries=batch)
    time.sleep(0.1)  # 100ms between batches
```

---

## Common Patterns

### Transactional Emails

```python
# Order confirmation
client.send_email(
    to=customer_email,
    template_name='order-confirmation',
    template_data={
        'customerName': customer.name,
        'orderId': order.id,
        'items': order.items,
        'total': order.total,
    },
    metadata={'orderId': order.id, 'type': 'transactional'},
)
```

### Password Reset

```python
# Password reset with security sender
client.send_email(
    to=user_email,
    template_name='password-reset',
    template_data={
        'userName': user.name,
        'resetLink': f'https://myapp.com/reset/{token}',
        'expiresIn': '24 hours',
    },
    sender_email='security@myapp.com',
    sender_name='MyApp Security',
    metadata={'userId': user.id, 'type': 'security'},
)
```

### Marketing Campaign

```python
# Newsletter with campaign tracking
for subscriber in subscribers:
    client.send_email(
        to=subscriber.email,
        template_name='newsletter-december',
        template_data={
            'firstName': subscriber.first_name,
            'unsubscribeLink': f'https://myapp.com/unsubscribe/{subscriber.id}',
        },
        metadata={
            'campaignId': 'newsletter-dec-2024',
            'subscriberId': subscriber.id,
            'segment': subscriber.segment,
        },
    )
```

---

## Receiving Status Notifications

SESEmailEngine publishes email status change events to EventBridge, allowing your services to receive real-time notifications about delivery, bounces, complaints, and opens.

### How It Works

When an email status changes (delivered, bounced, complained, opened), the Feedback Processor publishes an "Email Status Changed" event to the same EventBridge bus used for sending emails. Your services can create EventBridge rules to subscribe to these events.

```
Your App → EventBridge → SESEmailEngine → SES → Recipient
                ↑                           ↓
                └──── Status Notifications ←┘
```

### Status Change Event Schema

**Source**: `sesmailengine`
**Detail Type**: `Email Status Changed`

```json
{
  "source": "sesmailengine",
  "detail-type": "Email Status Changed",
  "detail": {
    "emailId": "email-abc123",
    "status": "delivered",
    "toEmail": "recipient@example.com",
    "timestamp": "2024-12-14T10:31:00Z",
    "originalSource": "my.application",
    "templateName": "welcome",
    "subject": "Welcome to Our Service",
    "sesMessageId": "0000014a-f896-4c47-b8da-f6f2e7b2c19a-000000",
    "metadata": {
      "campaignId": "welcome-2024",
      "userId": "user-456"
    }
  }
}
```

### Status-Specific Fields

**For bounces (`status: "bounced"` or `status: "soft_bounced"`):**
```json
{
  "status": "bounced",
  "bounceType": "Permanent",
  "bounceSubType": "NoEmail",
  "bounceReason": "NoEmail"
}
```

**For complaints (`status: "complained"`):**
```json
{
  "status": "complained",
  "complaintFeedbackType": "abuse"
}
```

### Creating EventBridge Rules

Create rules in your AWS account to subscribe to status notifications:

**Subscribe to all status changes:**
```json
{
  "source": ["sesmailengine"],
  "detail-type": ["Email Status Changed"]
}
```

**Subscribe to bounces only:**
```json
{
  "source": ["sesmailengine"],
  "detail-type": ["Email Status Changed"],
  "detail": {
    "status": ["bounced", "soft_bounced"]
  }
}
```

**Subscribe to events from your service only:**
```json
{
  "source": ["sesmailengine"],
  "detail-type": ["Email Status Changed"],
  "detail": {
    "originalSource": ["my.application"]
  }
}
```

### AWS CLI Example

Create a rule to send bounce notifications to your Lambda:

```bash
# Create the rule
aws events put-rule \
  --name "my-app-bounce-notifications" \
  --event-bus-name "sesmailengine-EmailBus" \
  --event-pattern '{
    "source": ["sesmailengine"],
    "detail-type": ["Email Status Changed"],
    "detail": {
      "status": ["bounced"],
      "originalSource": ["my.application"]
    }
  }'

# Add your Lambda as target
aws events put-targets \
  --rule "my-app-bounce-notifications" \
  --event-bus-name "sesmailengine-EmailBus" \
  --targets "Id"="1","Arn"="arn:aws:lambda:us-east-1:123456789012:function:my-bounce-handler"
```

### IAM Policy for Creating Rules

Your service needs permission to create rules on the SESEmailEngine event bus:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "events:PutRule",
        "events:PutTargets",
        "events:DeleteRule",
        "events:RemoveTargets",
        "events:DescribeRule"
      ],
      "Resource": "arn:aws:events:REGION:ACCOUNT_ID:rule/STACK_NAME-EmailBus/*"
    }
  ]
}
```

### Python Handler Example

Handle status notifications in your Lambda:

```python
def lambda_handler(event, context):
    detail = event['detail']
    
    email_id = detail['emailId']
    status = detail['status']
    to_email = detail['toEmail']
    metadata = detail.get('metadata', {})
    
    if status == 'bounced':
        bounce_type = detail.get('bounceType')
        bounce_reason = detail.get('bounceReason')
        print(f"Hard bounce for {to_email}: {bounce_reason}")
        # Update your user database, remove from mailing list, etc.
        
    elif status == 'soft_bounced':
        print(f"Soft bounce for {to_email}, will retry automatically")
        
    elif status == 'complained':
        print(f"Spam complaint from {to_email}")
        # Immediately stop sending to this user
        
    elif status == 'delivered':
        print(f"Email {email_id} delivered to {to_email}")
        
    elif status == 'opened':
        print(f"Email {email_id} opened by {to_email}")
    
    return {'statusCode': 200}
```

### Best Practices

1. **Filter by originalSource**: Only subscribe to events from your own service to avoid processing other services' notifications.

2. **Handle idempotently**: Status events may be delivered more than once. Use `emailId` to deduplicate.

3. **Don't block on notifications**: Status notifications are best-effort. Don't rely on them for critical business logic - query the tracking table for authoritative status.

4. **Use metadata for correlation**: Include `campaignId`, `userId`, or other identifiers in metadata when sending emails to correlate notifications with your business entities.

---

## Performance Benchmarks

SESEmailEngine is designed for high-throughput email sending. Below are performance characteristics based on load testing with AWS SES simulator addresses.

### Throughput

| Scenario | Throughput | Success Rate | Notes |
|----------|------------|--------------|-------|
| Light load (100 emails) | ~14 emails/sec | >99% | Matches SES rate limit |
| Standard load (1,000 emails) | ~14 emails/sec | >99% | Sustained rate |
| High load (10,000 emails) | ~14 emails/sec | >98% | Request SES increase for higher |

**Factors affecting throughput:**
- SES sending rate (default: 14/sec for new production accounts)
- Lambda concurrency (should match SES rate to avoid burst throttling)
- DynamoDB write capacity (on-demand scales automatically)
- Template complexity and size

### Latency

| Stage | P50 | P95 | P99 | Description |
|-------|-----|-----|-----|-------------|
| EventBridge Publish | 20-50ms | 80-150ms | 200-400ms | Time to publish event |
| Email Sender Lambda | 200-500ms | 800-1500ms | 2000-3000ms | Event → tracking record |
| Delivery Notification | 5-15s | 20-40s | 45-60s | SES → delivery confirmed |
| Suppression Creation | 5-15s | 20-40s | 45-60s | Bounce → suppression entry |

**Note:** Delivery notification latency depends on SES processing time and SNS delivery, which are outside SESEmailEngine's control.

### Cold Start Impact

Lambda cold starts can add 500-1500ms to the first request. To minimize cold starts:
- Use provisioned concurrency for consistent latency
- Keep Lambda warm with scheduled pings
- Accept occasional cold starts for cost efficiency

### Scaling Recommendations

Lambda concurrency should match your SES sending rate to avoid burst throttling.

| Daily Volume | Lambda Concurrency | SES Rate Limit | Notes |
|--------------|-------------------|----------------|-------|
| <1,200,000 | 14 (default) | 14/sec (default) | No changes needed |
| 1,200,000 - 4,300,000 | 50 | 50/sec | Request SES increase |
| >4,300,000 | 100+ | 100+/sec | Contact AWS Support |

**Calculation:** 14 emails/sec × 86,400 seconds/day = ~1.2 million emails/day

**To increase SES rate limit:**
1. Go to AWS SES Console → Account Dashboard
2. Click "Request production access" or "Request a sending limit increase"
3. After approval, update Lambda concurrency to match:
   ```bash
   aws cloudformation update-stack \
     --stack-name sesmailengine \
     --use-previous-template \
     --parameters ParameterKey=EmailSenderConcurrency,ParameterValue=50 \
     --capabilities CAPABILITY_NAMED_IAM
   ```

---

## Next Steps

- [TEMPLATES.md](TEMPLATES.md) - Create email templates
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) - Fix common errors
- [BOUNCE_RATE_PROTECTION.md](BOUNCE_RATE_PROTECTION.md) - Understand bounce protection
