# SESEmailEngine - Architecture Documentation

This document provides a detailed visualization and explanation of the SESEmailEngine infrastructure, based on the actual CloudFormation template and Lambda code.

---

## Deployment Architecture

SESMailEngine is distributed as a ZIP package that customers deploy to their own AWS account.

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              DISTRIBUTION PACKAGE                                       │
│                              sesmailengine-v1.0.0.zip                                   │
│                                                                                         │
│   ├── lambda/                      # Pre-packaged Lambda functions                      │
│   │   ├── email-sender.zip                                                              │
│   │   ├── feedback-processor.zip                                                        │
│   │   └── template-seeder.zip                                                           │
│   ├── starter-templates/           # 8 production-ready email templates                 │
│   ├── template.yaml                # CloudFormation template                            │
│   ├── install.py                   # Cross-platform Python installer                    │
│   ├── requirements.txt             # boto3 dependency                                   │
│   ├── README.md                    # Quick start guide                                  │
│   └── docs/                        # Documentation                                      │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                              │
                                              │ python install.py
                                              ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              CUSTOMER'S AWS ACCOUNT                                     │
│                                                                                         │
│   1. Creates S3 bucket for Lambda code and templates                                    │
│   2. Uploads Lambda ZIPs and starter templates to S3                                    │
│   3. Deploys CloudFormation stack                                                       │
│   4. Invokes Template Seeder to copy templates to customer bucket                       │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                    CUSTOMER APPLICATION                                 │
│                                                                                         │
│   aws events put-events --event-bus-name {StackName}-EmailBus                           │
│   --detail-type "Email Request"                                                         │
│   --detail '{"to":"...", "templateName":"...", "templateData":{...}}'                   │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                    AWS EVENTBRIDGE                                      │
│                                                                                         │
│   ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│   │  Custom Event Bus: {StackName}-EmailBus                                         │   │
│   │  Resource Policy: Allows account principals to PutEvents                        │   │
│   └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                              │                                          │
│   ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│   │  Event Rule: {StackName}-EmailEventRule                                         │   │
│   │  Pattern: detail-type = "Email Request"                                         │   │
│   │  Target: SQS Queue (EmailQueue)                                                 │   │
│   └─────────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                    AWS SQS                                              │
│                                                                                         │
│   ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│   │  Queue: {StackName}-EmailQueue                                                  │   │
│   │  ├── Visibility Timeout: 600s (10 minutes)                                      │   │
│   │  ├── Message Retention: 14 days                                                 │   │
│   │  ├── Long Polling: 20s                                                          │   │
│   │  └── DLQ: {StackName}-EmailQueueDLQ (after 3 attempts)                          │   │
│   └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
│   Event Source Mapping:                                                                 |  │   |                                                                                     │
│   └── MaximumConcurrency: 5 (configurable via EmailSenderConcurrency parameter)         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              EMAIL SENDER LAMBDA                                        │
│                              {StackName}-EmailSender                                    │
│                                                                                         │
│   Memory: 512MB (configurable)    Timeout: 30s                                          │
│   Concurrency: Controlled by SQS MaximumConcurrency (default: 5)                        │
│                                                                                         │
│   ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│   │  handler.py - Main entry point                                                  │   │
│   │                                                                                 │   │
│   │  Step 1: Parse event (SQS message containing EventBridge event)                 │   │
│   │  Step 2: Check for duplicate email ID (idempotency)                             │   │
│   │  Step 3: Check suppression list (DynamoDB)                                      │   │
│   │  Step 4: Load and render template (S3 + Jinja2)                                 │   │
│   │  Step 5: Resolve sender email (override → template → default)                   │   │
│   │  Step 6: Send email via SES                                                     │   │
│   │  Step 7: Track in DynamoDB                                                      │   │
│   └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
│   Services:                                                                             │
│   ├── suppression_service.py  - Check if email is suppressed                            │
│   ├── template_service.py     - Load templates from S3, render with Jinja2              │
│   ├── email_service.py        - Send via SES with configuration set                     │
│   ├── tracking_service.py     - Write tracking records to DynamoDB                      │
│   ├── error_handler.py        - Error categorization                                    │
│   └── utils.py                - Email validation, ID generation, env vars               │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                              │
                    ┌─────────────────────────┼─────────────────────────┐
                    │                         │                         │
                    ▼                         ▼                         ▼
┌───────────────────────────┐  ┌───────────────────────────┐  ┌───────────────────────────┐
│      DYNAMODB             │  │         S3                │  │         SES               │
│                           │  │                           │  │                           │
│  EmailTracking Table      │  │  Template Bucket          │  │  Configuration Set        │
│  ├── emailId (PK)         │  │  {StackName}-templates-   │  │  {StackName}-ConfigSet    │
│  ├── toEmail              │  │  {AccountId}              │  │                           │
│  ├── status               │  │                           │  │  Features:                │
│  ├── sesMessageId         │  │  Structure:               │  │  ├── TLS Required         │
│  ├── timestamp            │  │  templates/               │  │  ├── Reputation Metrics   │
│  ├── datePartition        │  │  ├── welcome/             │  │  └── Event Destinations   │
│  └── ttl                  │  │  │   ├── template.html    │  │      └── SNS Topic        │
│                           │  │  │   ├── template.txt     │  │                           │
│  GSIs:                    │  │  │   └── metadata.json    │  │  Events Published:        │
│  ├── to-email-timestamp   │  │  └── password-reset/      │  │  ├── bounce               │
│  ├── ses-message-id       │  │      └── ...              │  │  ├── complaint            │
│  └── date-partition       │  │                           │  │  ├── delivery             │
│                           │  │  Versioning: Enabled      │  │  ├── reject               │
│  Suppression Table        │  │  Encryption: AES256       │  │  └── open                 │
│  ├── email (PK)           │  │                           │  │                           │
│  ├── suppressionType      │  │  Public Access: Blocked   │  │                           │
│  ├── reason               │  │                           │  │                           │
│  └── addedToSuppressionDt │  │                           │  │                           │
│                           │  │                           │  │                           │
│  GSI: suppression-type-dt │  │                           │  │                           │
└───────────────────────────┘  └───────────────────────────┘  └───────────────────────────┘
                                                                          │
                                                                          │ SES Events
                                                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                    AWS SNS                                              │
│                                                                                         │
│   Topic: {StackName}-SESFeedback                                                        │
│   Policy: Allows ses.amazonaws.com to Publish                                           │
│                                                                                         │
│   Subscription: Lambda ({StackName}-FeedbackProcessor)                                  │
│   DLQ: {StackName}-FeedbackProcessorDLQ                                                 │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                           FEEDBACK PROCESSOR LAMBDA                                     │
│                           {StackName}-FeedbackProcessor                                 │
│                                                                                         │
│   Memory: 256MB (configurable)    Timeout: 15s    Concurrency: 100                      │
│                                                                                         │
│   ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│   │  handler.py - Main entry point                                                  │   │
│   │                                                                                 │   │
│   │  1. Parse SNS notification                                                      │   │
│   │  2. Extract notification type (bounce/complaint/delivery/open)                  │   │
│   │  3. Route to appropriate processor                                              │   │
│   │  4. Publish "Email Status Changed" event to EventBridge                         │   │
│   └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                         │
│   Processors:                                                                           │
│   ├── bounce_processor.py       - Hard bounce → suppress, Soft bounce → check consec.   │
│   ├── complaint_processor.py    - Add to suppression list                               │
│   ├── tracking_updater.py       - Update status (delivered, opened, bounced, etc.)      │
│   ├── suppression_updater.py    - Add entries to suppression table                      │
│   ├── notification_publisher.py - Publish status changes to EventBridge                 │
│   └── utils.py                  - SNS parsing, bounce type detection, status priority   │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                              │
                                              │ Status Notifications
                                              ▼
                              ┌───────────────────────────────┐
                              │       AWS EVENTBRIDGE         │
                              │                               │
                              │  "Email Status Changed"       │
                              │  events published to          │
                              │  {StackName}-EmailBus         │
                              │                               │
                              │  Consumer services create     │
                              │  rules to subscribe to        │
                              │  status updates               │
                              └───────────────────────────────┘
                                              │
                                              ▼
                              ┌───────────────────────────────┐
                              │   CUSTOMER SERVICES           │
                              │                               │
                              │   EventBridge rules filter    │
                              │   by status, originalSource   │
                              └───────────────────────────────┘
```

---

## Email Sending Flow (Happy Path)

```
1. Customer publishes event to EventBridge
   └── aws events put-events --event-bus-name {StackName}-EmailBus
       --detail-type "Email Request"
       --detail '{"to":"user@example.com","templateName":"welcome","templateData":{"userName":"John"}}'

2. EventBridge rule matches "Email Request" and sends to SQS EmailQueue

3. SQS triggers Email Sender Lambda (MaximumConcurrency controls throughput)

4. Email Sender Lambda (handler.py):
   a. parse_event() - Extract email data from SQS-wrapped EventBridge event
   b. suppression_service.check_suppression() - Query Suppression table
   c. template_service.get_template() - Load from S3 (with 5-min cache)
   d. template_service.render_template() - Render with Jinja2
   e. email_service.resolve_sender() - Priority: senderOverride → template → default
   f. email_service.send_email() - Call SES SendEmail API
   g. tracking_service.track_email_sent() - Write to EmailTracking table

5. SES sends email and publishes events to SNS topic

6. SNS triggers Feedback Processor Lambda

7. Feedback Processor Lambda (handler.py):
   a. parse_sns_message() - Extract SES notification
   b. Route to processor based on notification type:
      - delivery → tracking_updater.update_delivery()
      - open → tracking_updater.update_open()
      - bounce → bounce_processor.process_bounce()
      - complaint → complaint_processor.process_complaint()
```

---

## Bounce Handling Flow

### Hard Bounce (Permanent)
```
SES → SNS → Feedback Processor
         │
         ├── bounce_processor.process_bounce()
         │   ├── is_hard_bounce() returns True for:
         │   │   - bounceType="Permanent"
         │   │   - bounceSubType="NoEmail" or "Suppressed"
         │   │
         │   ├── tracking_updater.update_hard_bounce()
         │   │   └── Update status to "bounced" in EmailTracking
         │   │
         │   └── suppression_updater.add_hard_bounce()
         │       └── Add to Suppression table with reason="hard-bounce"
```

### Soft Bounce (Transient)
```
SES → SNS → Feedback Processor
         │
         ├── bounce_processor.process_bounce()
         │   ├── is_soft_bounce() returns True for:
         │   │   - bounceType="Transient"
         │   │   - bounceSubType="MailboxFull", "MessageTooLarge", etc.
         │   │
         │   ├── tracking_updater.update_soft_bounce()
         │   │   └── Update status to "soft_bounced" in EmailTracking
         │   │
         │   ├── _check_consecutive_soft_bounces() - Query last N emails to this address
         │   │
         │   └── If last N emails ALL have status "soft_bounced":
         │       └── suppression_updater.add_consecutive_soft_bounces()
         │           └── Add to Suppression with reason="consecutive-soft-bounces"
         │
         │   Note: N is configurable via ConsecutiveSoftBounceThreshold parameter (default: 3)
         │   Note: NO retry scheduling - AWS SES handles retries internally
```

---

## Dead Letter Queues (DLQs)

| DLQ Name | Source | Trigger Condition |
|----------|--------|-------------------|
| `{StackName}-EventBridgeDLQ` | EventBridge Rule | EventBridge fails to deliver to SQS |
| `{StackName}-EmailQueueDLQ` | EmailQueue | Email message fails after 3 receive attempts |
| `{StackName}-FeedbackProcessorDLQ` | SNS Subscription | Feedback Processor Lambda fails |

---

## CloudWatch Alarms

| Alarm Name | Metric | Threshold | Action |
|------------|--------|-----------|--------|
| `{StackName}-EmailQueueDLQ-Depth` | ApproximateNumberOfMessagesVisible | > 0 | SNS → AdminEmail |
| `{StackName}-FeedbackDLQ-Depth` | ApproximateNumberOfMessagesVisible | > 0 | SNS → AdminEmail |
| `{StackName}-EmailSender-Errors` | Lambda Errors | > 50 for 2 consecutive 5-min periods | SNS → AdminEmail |
| `{StackName}-FeedbackProcessor-Errors` | Lambda Errors | > 50 for 2 consecutive 5-min periods | SNS → AdminEmail |
| `{StackName}-SES-BounceRate` | SES Reputation.BounceRate | > 3% for 2 consecutive 1-hour periods | SNS → AdminEmail |
| `{StackName}-SES-ComplaintRate` | SES Reputation.ComplaintRate | > 0.05% for 2 consecutive 1-hour periods | SNS → AdminEmail |

**SES Reputation Alarms:** These alarms monitor your SES account's reputation metrics directly from AWS SES. The bounce rate alarm triggers at 3% (early warning before AWS's 5% review threshold). The complaint rate alarm triggers at 0.05% (early warning before AWS's 0.1% review threshold). Both use 2 consecutive 1-hour evaluation periods to reduce noise.See [BOUNCE_RATE_PROTECTION.md](BOUNCE_RATE_PROTECTION.md) for details.

**Lambda Error Alarm Design:** The threshold (>50 errors for 2 consecutive periods) is designed to ignore temporary burst throttling while catching sustained problems. A marketing campaign burst won't trigger false alarms, but real issues lasting 10+ minutes will alert you.

Alarm notifications are sent to SNS topic `{StackName}-Alarms`, which has an email subscription to the `AdminEmail` parameter.

---

## IAM Permissions Summary

### Email Sender Lambda Role
- **CloudWatch Logs**: CreateLogGroup, CreateLogStream, PutLogEvents
- **DynamoDB EmailTracking**: PutItem, UpdateItem, GetItem, Query (+ GSI indexes)
- **DynamoDB Suppression**: GetItem, Query (read-only)
- **S3 Template Bucket**: GetObject, ListBucket (prefix: templates/*)
- **SES**: SendEmail, SendRawEmail (identity/* and configuration-set)
- **SQS EmailQueue**: ReceiveMessage, DeleteMessage, GetQueueAttributes

### Feedback Processor Lambda Role
- **CloudWatch Logs**: CreateLogGroup, CreateLogStream, PutLogEvents
- **DynamoDB EmailTracking**: UpdateItem, Query (+ GSI indexes)
- **DynamoDB Suppression**: PutItem, UpdateItem, GetItem, Query
- **EventBridge**: PutEvents (for status notifications)

---

## Environment Variables

### Email Sender Lambda
| Variable | Source | Description |
|----------|--------|-------------|
| `TRACKING_TABLE` | EmailTrackingTable | DynamoDB table name |
| `SUPPRESSION_TABLE` | SuppressionTable | DynamoDB table name |
| `TEMPLATE_BUCKET` | TemplateBucket | S3 bucket name |
| `SES_CONFIGURATION_SET` | SESConfigurationSet | SES config set name |
| `DEFAULT_SENDER_EMAIL` | Parameter | Fallback sender email |
| `DEFAULT_SENDER_NAME` | Parameter | Fallback sender name |
| `DATA_RETENTION_DAYS` | Parameter | TTL for tracking records |
| `ENVIRONMENT` | Parameter | dev/staging/prod |
| `LOG_LEVEL` | Hardcoded | INFO |

### Feedback Processor Lambda
| Variable | Source | Description |
|----------|--------|-------------|
| `TRACKING_TABLE` | EmailTrackingTable | DynamoDB table name |
| `SUPPRESSION_TABLE` | SuppressionTable | DynamoDB table name |
| `CONSECUTIVE_SOFT_BOUNCE_THRESHOLD` | Parameter | Consecutive soft bounces before suppression (default: 3) |
| `ENVIRONMENT` | Parameter | dev/staging/prod |
| `LOG_LEVEL` | Hardcoded | INFO |

---

## Status Values and Transitions

```
                    ┌─────────────────────────────────────────────────────────┐
                    │                    EMAIL LIFECYCLE                      │
                    └─────────────────────────────────────────────────────────┘

    ┌─────────┐
    │  sent   │ ─────────────────────────────────────────────────────────────┐
    └────┬────┘                                                              │
         │                                                                   │
         ├──────────────────┬──────────────────┬──────────────────┐          │
         ▼                  ▼                  ▼                  ▼          │
    ┌─────────┐       ┌───────────┐      ┌──────────┐      ┌──────────┐      │
    │delivered│       │soft_bounce│      │ bounced  │      │complained│      │
    └────┬────┘       └───────────┘      └──────────┘      └──────────┘      │
         │                                                                   │
         ▼                                                                   │
    ┌─────────┐                                                              │
    │ opened  │ ◄────────────────────────────────────────────────────────────┘
    └─────────┘

    ┌─────────┐
    │ failed  │  (suppressed, template error, validation error)
    └─────────┘

Status Priority (higher number = higher priority, prevents out-of-order overwrites):
  sent(1) < soft_bounced(2) < delivered(3) < opened(4) < failed(5) < bounced/complained(6)
```

---

## Template Structure

```
s3://{StackName}-templates-{AccountId}/
└── templates/
    └── {templateName}/
        ├── template.html      # HTML body (Jinja2)
        ├── template.txt       # Plain text body (Jinja2)
        └── metadata.json      # Subject, sender, variables
```

### metadata.json Example
```json
{
  "subject": "Welcome to {{ companyName }}!",
  "senderName": "{{ companyName }} Team",
  "senderEmail": "{{ senderEmail | default('') }}",
  "description": "Welcome email for new users",
  "requiredVariables": ["userName", "companyName"],
  "optionalVariables": ["senderEmail"]
}
```

---

## Sender Email Resolution Priority

1. **senderOverride** in EventBridge event (highest priority)
   ```json
   {"senderOverride": {"email": "custom@example.com", "name": "Custom Name"}}
   ```

2. **Template metadata.json** senderEmail (supports Jinja2 variables)
   ```json
   {"senderEmail": "{{ senderEmail | default('noreply@example.com') }}"}
   ```

3. **DefaultSenderEmail** CloudFormation parameter (fallback)

---

*Document generated from actual CloudFormation template and Lambda code. Last updated: January 2026*


---

## Lambda Configuration

### Email Sender Lambda
| Setting | Default | Configurable | Notes |
|---------|---------|--------------|-------|
| Memory | 512MB | Yes | CloudFormation parameter |
| Timeout | 30s | Yes | CloudFormation parameter |
| Concurrency | Controlled by SQS | Yes | Via EmailSenderConcurrency parameter (MaximumConcurrency) |

### Feedback Processor Lambda
| Setting | Default | Configurable | Notes |
|---------|---------|--------------|-------|
| Memory | 256MB | Yes | CloudFormation parameter |
| Timeout | 15s | Yes | CloudFormation parameter |
| Concurrency | 100 | No | Fixed - handles SES feedback events |

### SQS EmailQueue → Email Sender
| Setting | Value | Notes |
|---------|-------|-------|
| BatchSize | 1 | **Critical**: Handler only processes `Records[0]`. BatchSize > 1 would cause data loss. |
| MaxBatchingWindow | 0 | No batching delay - process immediately |
| MaximumConcurrency | 5 (default) | Controls Lambda concurrency via SQS event source mapping |

---

## Performance Characteristics

SESMailEngine is designed for high throughput with zero data loss.

### Throughput

| Phase | Capability |
|-------|------------|
| Event Submission | 400-500 events/second (EventBridge) |
| Email Processing | Scales with your SES sending rate |

**SES Rate Limits by Tier:**
| Tier | Rate Limit | Daily Capacity |
|------|------------|----------------|
| Sandbox | 1-14/sec | ~1.2M emails |
| Production (typical) | 50-200/sec | 4-17M emails |
| Production (high volume) | 500+/sec | 43M+ emails |

### Data Integrity

- **Zero data loss architecture** - Every email is tracked in DynamoDB
- **Automatic retries** - EventBridge retries failed Lambda invocations
- **Dead letter queues** - Failed messages are preserved for investigation

### Cost Estimate (with AWS Free Tier)

Most services fall within AWS Free Tier for typical usage:

| Monthly Volume | SES | Lambda | DynamoDB | EventBridge | Total |
|----------------|-----|--------|----------|-------------|-------|
| 10,000 emails | $0 | $0 | $0 | $0.01 | **~$0.01** |
| 62,000 emails | $0 | $0 | $0 | $0.06 | **~$0.06** |
| 100,000 emails | $3.80 | $0 | $0 | $0.10 | **~$3.90** |

**Free Tier Allowances (monthly):**
- SES: 3000 emails when sent from Lambda/EC2
- Lambda: 1M requests + 400,000 GB-seconds
- DynamoDB: 25 GB storage + 25 WCU/RCU
- S3: 5 GB storage + 20,000 GET requests

**Comparison to SaaS alternatives:**
| Provider | 100k emails/month |
|----------|-------------------|
| SESMailEngine | ~$3.90 |
| SendGrid | $20+ |
| Mailgun | $35+ |
