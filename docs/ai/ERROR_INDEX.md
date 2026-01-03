# Error Index - AI Agent Reference

Machine-parseable error catalog for pattern matching and diagnosis.

## Error Format

Each error entry contains:
- **Code**: Unique identifier for AI reference
- **Pattern**: Regex pattern to match error message
- **Severity**: `permanent` (event consumed) or `retryable` (will retry)
- **Tracked**: Whether a DynamoDB record is created
- **Category**: Error type for grouping
- **Doc**: Link to user-facing documentation section

---

## Template Errors (TMPL)

### TMPL001
- **Pattern**: `Template '(.+)' not found`
- **Severity**: permanent
- **Tracked**: yes (status: failed)
- **Category**: template
- **Cause**: Template folder doesn't exist in S3 bucket
- **Doc**: TROUBLESHOOTING.md#template-x-not-found

### TMPL002
- **Pattern**: `Template '(.+)' missing template\.html`
- **Severity**: permanent
- **Tracked**: yes (status: failed)
- **Category**: template
- **Cause**: HTML file missing from template folder
- **Doc**: TROUBLESHOOTING.md#template-x-missing-templatehtml

### TMPL003
- **Pattern**: `Template '(.+)' missing template\.txt`
- **Severity**: permanent
- **Tracked**: yes (status: failed)
- **Category**: template
- **Cause**: Text file missing from template folder
- **Doc**: TROUBLESHOOTING.md#template-x-missing-templatetxt

### TMPL004
- **Pattern**: `Template '(.+)' missing metadata\.json`
- **Severity**: permanent
- **Tracked**: yes (status: failed)
- **Category**: template
- **Cause**: Metadata file missing from template folder
- **Doc**: TROUBLESHOOTING.md#template-x-missing-metadatajson

### TMPL005
- **Pattern**: `Template '(.+)' has invalid metadata\.json`
- **Severity**: permanent
- **Tracked**: yes (status: failed)
- **Category**: template
- **Cause**: JSON syntax error in metadata.json
- **Doc**: TROUBLESHOOTING.md#template-x-has-invalid-metadatajson

### TMPL006
- **Pattern**: `Missing required variables: \[(.+)\]`
- **Severity**: permanent
- **Tracked**: yes (status: failed)
- **Category**: template
- **Cause**: templateData missing variables listed in requiredVariables
- **Doc**: TROUBLESHOOTING.md#missing-required-variables

### TMPL007
- **Pattern**: `Undefined variable: '(.+)' is undefined`
- **Severity**: permanent
- **Tracked**: yes (status: failed)
- **Category**: template
- **Cause**: Template uses variable not provided in templateData
- **Doc**: TROUBLESHOOTING.md#undefined-variable

### TMPL008
- **Pattern**: `Template '(.+)' file too large: (\d+) bytes`
- **Severity**: permanent
- **Tracked**: yes (status: failed)
- **Category**: template
- **Cause**: Template file exceeds 256KB limit
- **Doc**: TROUBLESHOOTING.md#template-file-too-large

---

## Sender Errors (SNDR)

### SNDR001
- **Pattern**: `No sender email configured`
- **Severity**: permanent
- **Tracked**: yes (status: failed)
- **Category**: sender
- **Cause**: No sender in senderOverride, template metadata, or DefaultSenderEmail
- **Doc**: TROUBLESHOOTING.md#no-sender-email-configured

---

## Suppression Errors (SUPP)

### SUPP001
- **Pattern**: `Email not sent: recipient is suppressed \(hard-bounce\)`
- **Severity**: permanent
- **Tracked**: yes (status: failed)
- **Category**: suppression
- **Cause**: Recipient had permanent delivery failure
- **Doc**: TROUBLESHOOTING.md#email-not-sent-recipient-is-suppressed

### SUPP002
- **Pattern**: `Email not sent: recipient is suppressed \(soft-bounce-exceeded\)`
- **Severity**: permanent
- **Tracked**: yes (status: failed)
- **Category**: suppression
- **Cause**: Recipient had 15+ soft bounces in 30 days (cross-campaign protection)
- **Doc**: TROUBLESHOOTING.md#email-not-sent-recipient-is-suppressed

### SUPP003
- **Pattern**: `Email not sent: recipient is suppressed \(spam-complaint\)`
- **Severity**: permanent
- **Tracked**: yes (status: failed)
- **Category**: suppression
- **Cause**: Recipient marked email as spam
- **Doc**: TROUBLESHOOTING.md#email-not-sent-recipient-is-suppressed

---

## Bounce Rate Errors (RATE)

### RATE001
- **Pattern**: `Email not sent: bounce rate exceeded`
- **Severity**: permanent
- **Tracked**: yes (status: failed)
- **Category**: bounce_rate
- **Cause**: Daily bounce rate exceeds configured threshold
- **Doc**: TROUBLESHOOTING.md#email-not-sent-bounce-rate-exceeded

### RATE002
- **Pattern**: `Bounce rate exceeded during retry: [\d.]+%`
- **Severity**: permanent
- **Tracked**: yes (status: failed)
- **Category**: bounce_rate
- **Cause**: SQS retry blocked because daily bounce rate exceeds threshold
- **Doc**: BOUNCE_RATE_PROTECTION.md#sqs-retry-events-soft-bounce-retries

---

## SES Errors (SES)

### SES001
- **Pattern**: `SES error \(MessageRejected\)`
- **Severity**: permanent
- **Tracked**: yes (status: failed)
- **Category**: ses
- **Cause**: Sender email/domain not verified in SES
- **Doc**: TROUBLESHOOTING.md#messagerejected

### SES002
- **Pattern**: `SES error \(MailFromDomainNotVerified\)`
- **Severity**: permanent
- **Tracked**: yes (status: failed)
- **Category**: ses
- **Cause**: Custom MAIL FROM domain not verified
- **Doc**: TROUBLESHOOTING.md#mailfromdomainnotverified

### SES003
- **Pattern**: `SES error \(ConfigurationSetDoesNotExist\)`
- **Severity**: permanent
- **Tracked**: yes (status: failed)
- **Category**: ses
- **Cause**: SES configuration set missing or deleted
- **Doc**: TROUBLESHOOTING.md#configurationsetdoesnotexist

### SES004
- **Pattern**: `SES error \(AccountSendingPaused\)`
- **Severity**: permanent
- **Tracked**: yes (status: failed)
- **Category**: ses
- **Cause**: AWS paused SES sending due to reputation issues
- **Doc**: TROUBLESHOOTING.md#accountsendingpaused

### SES005
- **Pattern**: `SES error \(Throttling\)|Maximum sending rate exceeded`
- **Severity**: retryable
- **Tracked**: no (will retry)
- **Category**: ses
- **Cause**: Sending faster than SES quota allows
- **Note**: If all retries fail (Lambda internal + EventBridge), message goes to EventBridge DLQ with NO tracking record. Check DLQ for throttled emails.
- **Doc**: TROUBLESHOOTING.md#throttling--toomanyrequestsexception

### SES006
- **Pattern**: `Maximum sending rate exceeded`
- **Severity**: retryable (expected during bursts)
- **Tracked**: no (will retry)
- **Category**: ses
- **Cause**: Burst load exceeded SES rate limit - system will self-correct via retries
- **Note**: This is EXPECTED during high-volume sends. If all emails eventually deliver and DLQ is empty, no action needed. Only investigate if emails are missing or DLQ has messages.
- **Doc**: TROUBLESHOOTING.md#throttling-during-burst-loads-expected-behavior

---

## Validation Errors (VAL)

### VAL001
- **Pattern**: `Missing required field: 'to'`
- **Severity**: permanent
- **Tracked**: no (parsing failed)
- **Category**: validation
- **Cause**: EventBridge event missing recipient email
- **Doc**: TROUBLESHOOTING.md#missing-required-field-to

### VAL002
- **Pattern**: `Missing required field: 'templateName'`
- **Severity**: permanent
- **Tracked**: no (parsing failed)
- **Category**: validation
- **Cause**: EventBridge event missing template name
- **Doc**: TROUBLESHOOTING.md#missing-required-field-templatename

### VAL003
- **Pattern**: `Invalid email address format: (.+)`
- **Severity**: permanent
- **Tracked**: yes (status: failed)
- **Category**: validation
- **Cause**: Invalid email address in 'to' field
- **Doc**: TROUBLESHOOTING.md#invalid-email-address-format

### VAL004
- **Pattern**: `No subject provided`
- **Severity**: permanent
- **Tracked**: yes (status: failed)
- **Category**: validation
- **Cause**: No subject in event or template metadata
- **Doc**: TROUBLESHOOTING.md#no-subject-provided

---

## AWS Service Errors (AWS)

### AWS001
- **Pattern**: `DynamoDB error.*ProvisionedThroughputExceededException`
- **Severity**: retryable
- **Tracked**: no (will retry)
- **Category**: aws
- **Cause**: DynamoDB throttling (unusual with on-demand)
- **Doc**: TROUBLESHOOTING.md#dynamodb-errors

### AWS002
- **Pattern**: `S3 error.*AccessDenied`
- **Severity**: permanent
- **Tracked**: yes (status: failed)
- **Category**: aws
- **Cause**: Lambda missing S3 permissions
- **Doc**: TROUBLESHOOTING.md#s3-errors

---

## Feedback Processor Errors (FBK)

### FBK001
- **Pattern**: `Failed to parse SNS message`
- **Severity**: permanent
- **Tracked**: no (parsing failed)
- **Category**: feedback
- **Cause**: Invalid SNS notification format
- **Doc**: TROUBLESHOOTING.md#feedback-processor-errors

### FBK002
- **Pattern**: `Unknown notification type: (.+)`
- **Severity**: permanent
- **Tracked**: no (unknown type)
- **Category**: feedback
- **Cause**: SES sent unrecognized notification type
- **Doc**: TROUBLESHOOTING.md#feedback-processor-errors

### FBK003
- **Pattern**: `Failed to update tracking record`
- **Severity**: retryable
- **Tracked**: no (update failed)
- **Category**: feedback
- **Cause**: DynamoDB update failed
- **Doc**: TROUBLESHOOTING.md#feedback-processor-errors

### FBK004
- **Pattern**: `Failed to add to suppression list`
- **Severity**: retryable
- **Tracked**: no (update failed)
- **Category**: feedback
- **Cause**: DynamoDB suppression write failed
- **Doc**: TROUBLESHOOTING.md#feedback-processor-errors

### FBK005
- **Pattern**: `Failed to schedule retry`
- **Severity**: retryable
- **Tracked**: no (SQS failed)
- **Category**: feedback
- **Cause**: SQS send message failed
- **Doc**: TROUBLESHOOTING.md#feedback-processor-errors

### FBK006
- **Pattern**: `Email record not found for SES message ID`
- **Severity**: permanent
- **Tracked**: no (record missing)
- **Category**: feedback
- **Cause**: Tracking record doesn't exist for this SES message
- **Doc**: TROUBLESHOOTING.md#feedback-processor-errors

---

## Template Seeder Errors (SEED)

### SEED001
- **Pattern**: `Missing required environment variable: (.+)`
- **Severity**: permanent
- **Tracked**: no (configuration error)
- **Category**: seeder
- **Cause**: Lambda environment variable not configured
- **Doc**: TROUBLESHOOTING.md#template-seeder-errors

### SEED002
- **Pattern**: `No starter templates found in source bucket`
- **Severity**: permanent
- **Tracked**: no (no templates)
- **Category**: seeder
- **Cause**: Source bucket has no templates in starter-templates/ prefix
- **Doc**: TROUBLESHOOTING.md#template-seeder-errors

### SEED003
- **Pattern**: `AccessDenied.*sesmailengine-artifacts`
- **Severity**: permanent
- **Tracked**: no (permission error)
- **Category**: seeder
- **Cause**: Lambda missing S3 read permission on Marketplace bucket
- **Doc**: TROUBLESHOOTING.md#template-seeder-errors

### SEED004
- **Pattern**: `AccessDenied.*templates`
- **Severity**: permanent
- **Tracked**: no (permission error)
- **Category**: seeder
- **Cause**: Lambda missing S3 write permission on customer bucket
- **Doc**: TROUBLESHOOTING.md#template-seeder-errors

---

## Status Reference

| Status | Meaning | Final? |
|--------|---------|--------|
| sent | Accepted by SES | No |
| delivered | Confirmed delivery | Yes |
| soft_bounced | Temporary failure, will retry | No |
| bounced | Permanent failure | Yes |
| complained | Marked as spam | Yes |
| failed | Processing error | Yes |
| opened | Recipient opened | Yes |

**Status Priority (highest to lowest):**
opened > delivered > bounced > complained > soft_bounced > failed > sent

---

## CloudWatch Alarm Errors (ALM)

### ALM001
- **Pattern**: `ALARM: .+-EventBridgeDLQDepthAlarm`
- **Severity**: requires_attention
- **Tracked**: no (infrastructure alarm)
- **Category**: alarm
- **Cause**: Messages accumulating in EventBridge DLQ (failed Lambda invocations)
- **Doc**: TROUBLESHOOTING.md#dlq-depth-alarms

### ALM002
- **Pattern**: `ALARM: .+-RetryDLQDepthAlarm`
- **Severity**: requires_attention
- **Tracked**: no (infrastructure alarm)
- **Category**: alarm
- **Cause**: Messages accumulating in Retry DLQ (failed retry attempts)
- **Doc**: TROUBLESHOOTING.md#dlq-depth-alarms

### ALM003
- **Pattern**: `ALARM: .+-FeedbackDLQDepthAlarm`
- **Severity**: requires_attention
- **Tracked**: no (infrastructure alarm)
- **Category**: alarm
- **Cause**: Messages accumulating in Feedback DLQ (failed notification processing)
- **Doc**: TROUBLESHOOTING.md#dlq-depth-alarms

### ALM004
- **Pattern**: `ALARM: .+-EmailSenderErrorsAlarm`
- **Severity**: requires_attention
- **Tracked**: no (infrastructure alarm)
- **Category**: alarm
- **Cause**: Email Sender Lambda throwing errors
- **Doc**: TROUBLESHOOTING.md#lambda-error-alarms

### ALM005
- **Pattern**: `ALARM: .+-FeedbackProcessorErrorsAlarm`
- **Severity**: requires_attention
- **Tracked**: no (infrastructure alarm)
- **Category**: alarm
- **Cause**: Feedback Processor Lambda throwing errors
- **Doc**: TROUBLESHOOTING.md#lambda-error-alarms

### ALM006
- **Pattern**: `ALARM: .+-SESBounceRateAlarm`
- **Severity**: critical
- **Tracked**: no (infrastructure alarm)
- **Category**: alarm
- **Cause**: SES bounce rate exceeds 3% (AWS suspends at ~5%)
- **Doc**: TROUBLESHOOTING.md#ses-bounce-rate-alarm

### ALM007
- **Pattern**: `ALARM: .+-SESComplaintRateAlarm`
- **Severity**: critical
- **Tracked**: no (infrastructure alarm)
- **Category**: alarm
- **Cause**: SES complaint rate exceeds 0.05% (AWS suspends at ~0.1%)
- **Doc**: TROUBLESHOOTING.md#ses-complaint-rate-alarm
