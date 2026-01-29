# Diagnostic Queries - AI Agent Reference

AWS CLI queries for diagnosing email issues. Replace `{STACK}` with customer's stack name.

---

## Finding Stack Resources

### Get stack name from any resource
```bash
# If customer knows their EventBridge bus name
aws events describe-event-bus --name {BUS_NAME} --query 'Tags'

# List all SESMailEngine stacks
aws cloudformation list-stacks --query "StackSummaries[?contains(StackName, 'email') || contains(StackName, 'Email')].StackName"
```

### Get resource names from stack
```bash
aws cloudformation describe-stacks --stack-name {STACK} --query 'Stacks[0].Outputs'
```

**Key outputs:**
- `EmailTrackingTableName` - DynamoDB tracking table
- `SuppressionTableName` - DynamoDB suppression table
- `EventBusName` - EventBridge bus name
- `TemplateBucketName` - S3 template bucket

---

## Email Lookup Queries

### Find email by emailId (primary key)
```bash
aws dynamodb get-item \
  --table-name {STACK}-EmailTracking \
  --key '{"emailId": {"S": "EMAIL_ID_HERE"}}'
```

### Find email by SES message ID
```bash
aws dynamodb query \
  --table-name {STACK}-EmailTracking \
  --index-name ses-message-id-index \
  --key-condition-expression "sesMessageId = :id" \
  --expression-attribute-values '{":id": {"S": "SES_MESSAGE_ID_HERE"}}'
```

### Find all emails to a recipient
```bash
aws dynamodb query \
  --table-name {STACK}-EmailTracking \
  --index-name to-email-timestamp-index \
  --key-condition-expression "toEmail = :email" \
  --expression-attribute-values '{":email": {"S": "user@example.com"}}'
```

### Find recent emails to recipient (last 24h)
```bash
aws dynamodb query \
  --table-name {STACK}-EmailTracking \
  --index-name to-email-timestamp-index \
  --key-condition-expression "toEmail = :email AND #ts > :since" \
  --expression-attribute-names '{"#ts": "timestamp"}' \
  --expression-attribute-values '{":email": {"S": "user@example.com"}, ":since": {"S": "2024-12-18T00:00:00Z"}}'
```

### Find failed emails for recipient
```bash
aws dynamodb query \
  --table-name {STACK}-EmailTracking \
  --index-name to-email-timestamp-index \
  --key-condition-expression "toEmail = :email" \
  --filter-expression "#s = :status" \
  --expression-attribute-names '{"#s": "status"}' \
  --expression-attribute-values '{":email": {"S": "user@example.com"}, ":status": {"S": "failed"}}'
```

### Find all retry attempts for an email (using original-email-id-index GSI)
```bash
aws dynamodb query \
  --table-name {STACK}-EmailTracking \
  --index-name original-email-id-index \
  --key-condition-expression "originalEmailId = :origId" \
  --expression-attribute-values '{":origId": {"S": "ORIGINAL_EMAIL_ID"}}'
```

**Note:** This is the preferred method for tracking retry chains. It returns ONLY the attempts for a specific email, not all emails to the recipient.

### Find all retry attempts for an email (alternative: by recipient)
```bash
aws dynamodb query \
  --table-name {STACK}-EmailTracking \
  --index-name to-email-timestamp-index \
  --key-condition-expression "toEmail = :email" \
  --filter-expression "originalEmailId = :orig" \
  --expression-attribute-values '{":email": {"S": "user@example.com"}, ":orig": {"S": "ORIGINAL_EMAIL_ID"}}'
```

---

## Suppression Queries

### Check if email is suppressed
```bash
aws dynamodb get-item \
  --table-name {STACK}-Suppression \
  --key '{"email": {"S": "user@example.com"}}'
```

**If item exists:** Email is blocked. Check `reason` field:
- `hard-bounce` - Address doesn't exist
- `soft-bounce-exceeded` - 3+ temporary failures
- `spam-complaint` - User reported spam

### List recent suppressions (bounces)
```bash
aws dynamodb query \
  --table-name {STACK}-Suppression \
  --index-name suppression-type-date-index \
  --key-condition-expression "suppressionType = :type" \
  --expression-attribute-values '{":type": {"S": "bounce"}}' \
  --scan-index-forward false \
  --limit 20
```

### List recent complaints
```bash
aws dynamodb query \
  --table-name {STACK}-Suppression \
  --index-name suppression-type-date-index \
  --key-condition-expression "suppressionType = :type" \
  --expression-attribute-values '{":type": {"S": "complaint"}}' \
  --scan-index-forward false \
  --limit 20
```

### Remove email from suppression (use with caution!)
```bash
aws dynamodb delete-item \
  --table-name {STACK}-Suppression \
  --key '{"email": {"S": "user@example.com"}}'
```

---

## Bounce Rate Queries

### Count today's emails
```bash
TODAY=$(date -u +%Y-%m-%d)
aws dynamodb query \
  --table-name {STACK}-EmailTracking \
  --index-name date-partition-index \
  --key-condition-expression "datePartition = :date" \
  --expression-attribute-values "{\":date\": {\"S\": \"$TODAY\"}}" \
  --select COUNT
```

### Count today's bounces
```bash
TODAY=$(date -u +%Y-%m-%d)
aws dynamodb query \
  --table-name {STACK}-Suppression \
  --index-name suppression-type-date-index \
  --key-condition-expression "suppressionType = :type AND begins_with(addedToSuppressionDate, :date)" \
  --expression-attribute-values "{\":type\": {\"S\": \"bounce\"}, \":date\": {\"S\": \"$TODAY\"}}" \
  --select COUNT
```

### Calculate bounce rate
```bash
# Get counts from above queries, then:
# bounce_rate = bounce_count / email_count * 100
# If > 5% (default threshold), sending is blocked
```

---

## Template Queries

### List templates in S3
```bash
aws s3 ls s3://{BUCKET}/templates/ --recursive
```

### Check specific template exists
```bash
aws s3 ls s3://{BUCKET}/templates/{TEMPLATE_NAME}/
```

**Expected files:**
- `template.html`
- `template.txt`
- `metadata.json`

### View template metadata
```bash
aws s3 cp s3://{BUCKET}/templates/{TEMPLATE_NAME}/metadata.json -
```

### Validate metadata JSON syntax
```bash
aws s3 cp s3://{BUCKET}/templates/{TEMPLATE_NAME}/metadata.json - | python -m json.tool
```

---

## SES Queries

### Check sender verification status
```bash
aws ses get-identity-verification-attributes \
  --identities sender@example.com yourdomain.com
```

### Check SES sending quota
```bash
aws ses get-send-quota
```

### Check SES configuration set
```bash
aws ses describe-configuration-set \
  --configuration-set-name {STACK}-ConfigSet
```

### Check SES account status
```bash
aws ses get-account-sending-enabled
```

---

## CloudWatch Log Queries

### Find errors in Email Sender Lambda
```bash
aws logs filter-log-events \
  --log-group-name /aws/lambda/{STACK}-EmailSender \
  --filter-pattern "ERROR" \
  --start-time $(date -d '1 hour ago' +%s000) \
  --limit 50
```

### Find SES throttling errors (burst load investigation)
```bash
aws logs filter-log-events \
  --log-group-name /aws/lambda/{STACK}-EmailSender \
  --filter-pattern "Maximum sending rate exceeded" \
  --start-time $(date -d '1 hour ago' +%s000) \
  --limit 50
```

**Note:** Throttling errors during burst loads are EXPECTED behavior. The system retries automatically. Only investigate if:
- Emails are missing after all retries complete
- EventBridge DLQ has messages
- Throttling persists even with low volume

### Find specific email in logs
```bash
aws logs filter-log-events \
  --log-group-name /aws/lambda/{STACK}-EmailSender \
  --filter-pattern "EMAIL_ID_HERE" \
  --start-time $(date -d '24 hours ago' +%s000)
```

### Find template errors
```bash
aws logs filter-log-events \
  --log-group-name /aws/lambda/{STACK}-EmailSender \
  --filter-pattern "Template error" \
  --start-time $(date -d '1 hour ago' +%s000)
```

### Find suppression blocks
```bash
aws logs filter-log-events \
  --log-group-name /aws/lambda/{STACK}-EmailSender \
  --filter-pattern "suppressed" \
  --start-time $(date -d '1 hour ago' +%s000)
```

### Find bounce rate blocks
```bash
aws logs filter-log-events \
  --log-group-name /aws/lambda/{STACK}-EmailSender \
  --filter-pattern "bounce_rate_exceeded" \
  --start-time $(date -d '1 hour ago' +%s000)
```

### Find errors in Feedback Processor Lambda
```bash
aws logs filter-log-events \
  --log-group-name /aws/lambda/{STACK}-FeedbackProcessor \
  --filter-pattern "ERROR" \
  --start-time $(date -d '1 hour ago' +%s000)
```

### Find bounce processing events
```bash
aws logs filter-log-events \
  --log-group-name /aws/lambda/{STACK}-FeedbackProcessor \
  --filter-pattern "bounce" \
  --start-time $(date -d '1 hour ago' +%s000)
```

### Find complaint processing events
```bash
aws logs filter-log-events \
  --log-group-name /aws/lambda/{STACK}-FeedbackProcessor \
  --filter-pattern "complaint" \
  --start-time $(date -d '1 hour ago' +%s000)
```

---

## Template Seeder Queries

### Run Template Seeder Lambda
```bash
aws lambda invoke \
  --function-name {STACK}-TemplateSeeder \
  --output text /dev/stdout
```

**Expected response:**
```json
{
  "statusCode": 200,
  "installed": ["welcome", "password-reset"],
  "skipped": [],
  "message": "Installed 2 templates, skipped 0 existing"
}
```

### Check Template Seeder Lambda exists
```bash
aws lambda get-function \
  --function-name {STACK}-TemplateSeeder \
  --query 'Configuration.{Name:FunctionName,State:State,LastModified:LastModified}'
```

### Check Template Seeder logs
```bash
aws logs filter-log-events \
  --log-group-name /aws/lambda/{STACK}-TemplateSeeder \
  --start-time $(date -d '1 hour ago' +%s000) \
  --limit 50
```

### Check Template Seeder errors
```bash
aws logs filter-log-events \
  --log-group-name /aws/lambda/{STACK}-TemplateSeeder \
  --filter-pattern "ERROR" \
  --start-time $(date -d '1 hour ago' +%s000)
```

### Verify installed templates
```bash
# Get bucket name
BUCKET=$(aws cloudformation describe-stacks --stack-name {STACK} \
  --query 'Stacks[0].Outputs[?OutputKey==`TemplateBucketName`].OutputValue' --output text)

# List templates
aws s3 ls s3://$BUCKET/templates/
```

---

## Soft Bounce Queries

### Check consecutive soft bounces for a recipient
```bash
# Get last 3 emails to this address (most recent first)
aws dynamodb query \
  --table-name {STACK}-EmailTracking \
  --index-name to-email-timestamp-index \
  --key-condition-expression "toEmail = :email" \
  --expression-attribute-values '{":email": {"S": "user@example.com"}}' \
  --scan-index-forward false \
  --limit 3
```

**Note:** If all 3 most recent emails have status "soft_bounced", the address will be permanently suppressed.

### Find soft bounce history for an email address
```bash
aws dynamodb query \
  --table-name {STACK}-EmailTracking \
  --index-name to-email-timestamp-index \
  --key-condition-expression "toEmail = :email" \
  --filter-expression "#s = :status" \
  --expression-attribute-names '{"#s": "status"}' \
  --expression-attribute-values '{":email": {"S": "user@example.com"}, ":status": {"S": "soft_bounced"}}'
```

---

## DLQ Queries

### Check EventBridge DLQ depth
```bash
aws sqs get-queue-attributes \
  --queue-url https://sqs.{REGION}.amazonaws.com/{ACCOUNT}/{STACK}-EventBridgeDLQ \
  --attribute-names ApproximateNumberOfMessages
```

### Check EmailQueue DLQ depth
```bash
aws sqs get-queue-attributes \
  --queue-url https://sqs.{REGION}.amazonaws.com/{ACCOUNT}/{STACK}-EmailQueueDLQ \
  --attribute-names ApproximateNumberOfMessages
```

### Check Feedback Processor DLQ depth
```bash
aws sqs get-queue-attributes \
  --queue-url https://sqs.{REGION}.amazonaws.com/{ACCOUNT}/{STACK}-FeedbackProcessorDLQ \
  --attribute-names ApproximateNumberOfMessages
```

### View DLQ messages (without consuming)
```bash
aws sqs receive-message \
  --queue-url https://sqs.{REGION}.amazonaws.com/{ACCOUNT}/{STACK}-EventBridgeDLQ \
  --max-number-of-messages 5 \
  --visibility-timeout 0
```

### Check EventBridge DLQ for throttled emails
```bash
# Get DLQ URL from CloudFormation outputs
DLQ_URL=$(aws cloudformation describe-stacks \
  --stack-name {STACK} \
  --query 'Stacks[0].Outputs[?OutputKey==`EventBridgeDLQUrl`].OutputValue' \
  --output text)

# View messages (without consuming)
aws sqs receive-message \
  --queue-url $DLQ_URL \
  --max-number-of-messages 10 \
  --visibility-timeout 0

# Check message count
aws sqs get-queue-attributes \
  --queue-url $DLQ_URL \
  --attribute-names ApproximateNumberOfMessages
```

**Note:** Throttled emails that exhaust all retries (Lambda internal + EventBridge) end up in the EventBridge DLQ with NO tracking record in DynamoDB. This is the only place to find them.

---

## CloudWatch Alarm Queries

### List all alarms for the stack
```bash
aws cloudwatch describe-alarms \
  --alarm-name-prefix {STACK} \
  --query 'MetricAlarms[].{Name:AlarmName,State:StateValue,Reason:StateReason}'
```

### Check alarm state
```bash
aws cloudwatch describe-alarms \
  --alarm-names {STACK}-EventBridgeDLQDepthAlarm \
  --query 'MetricAlarms[0].{State:StateValue,Reason:StateReason,Timestamp:StateUpdatedTimestamp}'
```

### Get alarm history (state changes)
```bash
aws cloudwatch describe-alarm-history \
  --alarm-name {STACK}-SESBounceRateAlarm \
  --history-item-type StateUpdate \
  --start-date $(date -d '7 days ago' -Iseconds) \
  --end-date $(date -Iseconds)
```

### Check SES bounce rate metric
```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/SES \
  --metric-name Reputation.BounceRate \
  --start-time $(date -d '24 hours ago' -Iseconds) \
  --end-time $(date -Iseconds) \
  --period 3600 \
  --statistics Average
```

### Check SES complaint rate metric
```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/SES \
  --metric-name Reputation.ComplaintRate \
  --start-time $(date -d '24 hours ago' -Iseconds) \
  --end-time $(date -Iseconds) \
  --period 3600 \
  --statistics Average
```

### Check Lambda error count
```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Errors \
  --dimensions Name=FunctionName,Value={STACK}-EmailSender \
  --start-time $(date -d '1 hour ago' -Iseconds) \
  --end-time $(date -Iseconds) \
  --period 300 \
  --statistics Sum
```

### Get alarm notification topic
```bash
aws cloudformation describe-stacks --stack-name {STACK} \
  --query 'Stacks[0].Outputs[?OutputKey==`AlarmNotificationTopicArn`].OutputValue' \
  --output text
```

**Available alarms:**
- `{STACK}-EventBridgeDLQ-Depth` - Failed EventBridge invocations
- `{STACK}-FeedbackDLQ-Depth` - Failed feedback processing
- `{STACK}-EmailSenderErrorsAlarm` - Email Sender Lambda errors
- `{STACK}-FeedbackProcessorErrorsAlarm` - Feedback Processor Lambda errors
- `{STACK}-SESBounceRateAlarm` - SES bounce rate > 3%
- `{STACK}-SESComplaintRateAlarm` - SES complaint rate > 0.05%

---

## Status Notification Queries

### Check if status notifications are enabled
```bash
# Notifications are enabled if EVENT_BUS_NAME env var is set
aws lambda get-function-configuration \
  --function-name {STACK}-FeedbackProcessor \
  --query 'Environment.Variables.EVENT_BUS_NAME'
```

### Find status notification errors in logs
```bash
aws logs filter-log-events \
  --log-group-name /aws/lambda/{STACK}-FeedbackProcessor \
  --filter-pattern "Failed to publish status" \
  --start-time $(date -d '1 hour ago' +%s000)
```

### List EventBridge rules subscribed to status notifications
```bash
aws events list-rules \
  --event-bus-name {STACK}-EmailBus \
  --query 'Rules[?contains(EventPattern, `Email Status Changed`)].{Name:Name,State:State}'
```

### Test status notification event pattern
```bash
# Verify your rule pattern matches status events
aws events test-event-pattern \
  --event-pattern '{
    "source": ["sesmailengine"],
    "detail-type": ["Email Status Changed"],
    "detail": {"status": ["bounced"]}
  }' \
  --event '{
    "source": "sesmailengine",
    "detail-type": "Email Status Changed",
    "detail": {"status": "bounced", "emailId": "test-123"}
  }'
```
