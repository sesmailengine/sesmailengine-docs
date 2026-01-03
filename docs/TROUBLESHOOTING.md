# Troubleshooting Guide

This guide covers common errors you may encounter when using SESEmailEngine and how to resolve them.

## Quick Reference

| Error Type | Common Cause | Quick Fix |
|------------|--------------|-----------|
| Installer errors | Prerequisites or permissions | Check Python version and AWS credentials |
| Template errors | Missing files or variables | Check S3 bucket and templateData |
| Sender errors | No sender configured | Set DefaultSenderEmail or senderOverride |
| Suppression errors | Recipient bounced/complained | Remove from suppression or use different address |
| Bounce rate errors | Too many bounces | Clean your email list, wait for rate to recover |
| SES errors | Domain not verified or account issues | Check SES console |

---

## Installer Errors

### Python version not supported

**Error:** `Python 3.8 or higher is required. You have Python 3.6.`

**Fix:** Install Python 3.8 or higher:
- **macOS:** `brew install python@3.11`
- **Ubuntu:** `sudo apt install python3.11`
- **Windows:** Download from [python.org](https://python.org)

---

### AWS credentials not configured

**Error:** `AWS credentials not found. Please configure AWS CLI or set environment variables.`

**Fix:** Configure AWS credentials using one of these methods:

1. **AWS CLI (recommended):**
   ```bash
   aws configure
   ```

2. **Environment variables:**
   ```bash
   export AWS_ACCESS_KEY_ID=your-access-key
   export AWS_SECRET_ACCESS_KEY=your-secret-key
   export AWS_REGION=eu-west-2
   ```

3. **Verify credentials work:**
   ```bash
   aws sts get-caller-identity
   ```

---

### S3 bucket name already exists

**Error:** `S3 bucket 'my-bucket' already exists. S3 bucket names are globally unique.`

**When:** Someone else (anywhere in the world) already has a bucket with that name.

**Fix:** Choose a more unique bucket name:
- Add your company name: `acme-corp-sesmailengine`
- Add a random suffix: `my-company-sesmailengine-2024`
- Add your AWS account ID: `sesmailengine-123456789012`

---

### Insufficient IAM permissions

**Error:** `Access Denied: User is not authorized to perform cloudformation:CreateStack`

**When:** Your AWS credentials don't have admin permissions.

**Fix:** Use credentials with administrator access, or ensure your IAM user/role has these permissions:
- `cloudformation:*`
- `s3:*`
- `lambda:*`
- `dynamodb:*`
- `events:*`
- `sns:*`
- `sqs:*`
- `ses:*`
- `iam:*`
- `logs:*`
- `cloudwatch:*`

---

### Stack already exists

**Error:** `Stack 'sesmailengine' already exists.`

**Fix:** Either:
1. Use a different stack name: `python install.py` and enter a different name
2. Delete the existing stack first:
   ```bash
   aws cloudformation delete-stack --stack-name sesmailengine
   aws cloudformation wait stack-delete-complete --stack-name sesmailengine
   ```

---

### CloudFormation stack creation failed

**Error:** `Stack creation failed: CREATE_FAILED`

**When:** CloudFormation couldn't create one or more resources.

**Fix:**
1. Check the installer output for the specific failure reason
2. Check CloudFormation events in AWS Console:
   - Go to CloudFormation → Stacks → your-stack → Events
   - Look for `CREATE_FAILED` status and read the reason
3. Common causes:
   - **SES email not verified:** Verify your sender email in SES first
   - **Service limits:** Request limit increases in AWS Support
   - **Region not supported:** Try a different AWS region

---

### Email address not verified in SES

**Error:** `The sender email 'noreply@example.com' is not verified in SES.`

**When:** You're trying to use an email address that hasn't been verified in SES.

**Fix:**
1. Go to AWS SES Console → Verified identities
2. Click "Create identity"
3. Choose "Email address" and enter your email
4. Check your inbox and click the verification link
5. Run the installer again

**Note:** In SES sandbox mode, you must also verify recipient email addresses.

---

### SES Region Mismatch

**Error:** `Email address is not verified` or `Identity does not exist`

**When:** SESMailEngine is deployed in a different region than your SES verified identities.

**Why this happens:** SES is a regional service. Verified domains/emails in `us-east-1` are NOT available in `eu-west-1`.

**Fix:**
1. Check which region has your verified identities:
   ```bash
   # Check multiple regions
   aws ses list-identities --region us-east-1
   aws ses list-identities --region eu-west-1
   aws ses list-identities --region ap-southeast-1
   ```

2. Redeploy SESMailEngine to the correct region:
   ```bash
   # Uninstall from wrong region
   python install.py --uninstall
   
   # Set correct region
   export AWS_REGION=eu-west-1  # Your SES region
   
   # Reinstall
   python install.py
   ```

**Prevention:** The installer now prompts you to confirm the region matches your SES setup. Always verify before proceeding.

---

### Network connection error

**Error:** `Unable to connect to AWS. Check your internet connection.`

**Fix:**
1. Check your internet connection
2. If behind a corporate proxy, configure proxy settings:
   ```bash
   export HTTP_PROXY=http://proxy.company.com:8080
   export HTTPS_PROXY=http://proxy.company.com:8080
   ```
3. If using VPN, ensure AWS endpoints are accessible

---

### Installer log file

The installer creates a detailed log file for debugging:

```bash
cat sesmailengine-install.log
```

This contains:
- All AWS API calls made
- Full error messages and stack traces
- Timestamps for each operation

---

### Manual cleanup after failed install

If installation fails partway through, clean up before retrying:

```bash
# Delete partial CloudFormation stack (if created)
aws cloudformation delete-stack --stack-name sesmailengine
aws cloudformation wait stack-delete-complete --stack-name sesmailengine

# Delete S3 bucket (if created)
# First, empty the bucket
aws s3 rm s3://my-company-sesmailengine --recursive
# Then delete the bucket
aws s3 rb s3://my-company-sesmailengine
```

---

## Template Errors

### Template 'X' not found

**Error:** `Template 'welcome' not found`

**When:** The template folder doesn't exist in your S3 bucket.

**Fix:**
1. Verify the template exists in S3:
   ```bash
   aws s3 ls s3://${BUCKET}/templates/welcome/
   ```
2. Upload the template if missing:
   ```bash
   aws s3 cp welcome/ s3://${BUCKET}/templates/welcome/ --recursive
   ```

---

### Template 'X' missing template.html

**Error:** `Template 'welcome' missing template.html`

**When:** The template folder exists but is missing the HTML file.

**Fix:** Upload the missing file:
```bash
aws s3 cp template.html s3://${BUCKET}/templates/welcome/template.html
```

**Required files for each template:**
- `template.html` - HTML version
- `template.txt` - Plain text version
- `metadata.json` - Configuration

---

### Template 'X' missing template.txt

**Error:** `Template 'welcome' missing template.txt`

**When:** The template folder is missing the plain text version.

**Fix:** Create and upload a text version:
```bash
echo "Hello {{ userName }}" > template.txt
aws s3 cp template.txt s3://${BUCKET}/templates/welcome/template.txt
```

---

### Template 'X' missing metadata.json

**Error:** `Template 'welcome' missing metadata.json`

**When:** The template folder is missing the metadata configuration.

**Fix:** Create and upload metadata:
```json
{
  "subject": "Welcome!",
  "requiredVariables": ["userName"]
}
```
```bash
aws s3 cp metadata.json s3://${BUCKET}/templates/welcome/metadata.json
```

---

### Template 'X' has invalid metadata.json

**Error:** `Template 'welcome' has invalid metadata.json: Expecting property name...`

**When:** The metadata.json file has invalid JSON syntax.

**Fix:**
1. Download and check the file:
   ```bash
   aws s3 cp s3://${BUCKET}/templates/welcome/metadata.json .
   cat metadata.json | python -m json.tool
   ```
2. Fix JSON syntax errors (missing commas, quotes, brackets)
3. Re-upload the corrected file

---

### Missing required variables: ['userName', 'domain']

**Error:** `Failed to render template 'welcome': Missing required variables: ['userName', 'domain']`

**When:** Your `templateData` doesn't include all variables listed in the template's `requiredVariables`.

**Fix:** Add the missing variables to your EventBridge event:
```json
{
  "detail": {
    "templateName": "welcome",
    "templateData": {
      "userName": "John",
      "domain": "acme.com"
    }
  }
}
```

**Check required variables:**
```bash
aws s3 cp s3://${BUCKET}/templates/welcome/metadata.json - | jq '.requiredVariables'
```

---

### Undefined variable: 'X' is undefined

**Error:** `Failed to render template 'welcome': Undefined variable: 'companyName' is undefined`

**When:** Your template uses a variable that wasn't provided in `templateData` and isn't listed in `requiredVariables`.

**Fix:** Either:
1. Add the variable to your `templateData`
2. Or update `metadata.json` to include it in `requiredVariables` for clearer error messages

---

### Template syntax error

**Error:** `Failed to render template 'welcome': Syntax error: unexpected '}'`

**When:** Your Jinja2 template has syntax errors.

**Fix:** Check your template for:
- Unclosed `{{ }}` or `{% %}` tags
- Missing `{% endif %}` or `{% endfor %}`
- Typos in Jinja2 keywords

---

### Template file too large

**Error:** `Template 'welcome' file too large: 300000 bytes (max 262144 bytes)`

**When:** A template file exceeds the 256 KB limit.

**Fix:**
- Move images to a CDN and use URLs instead of inline base64
- Simplify CSS (don't embed entire frameworks)
- Split into multiple smaller templates

**Limit:** 256 KB per file (template.html, template.txt, metadata.json)

---

## Sender Configuration Errors

### No sender email configured

**Error:** `No sender email configured. Provide senderOverride in event, senderEmail in template metadata, or set DEFAULT_SENDER_EMAIL.`

**When:** No sender email is available from any of the three sources.

**Fix (choose one):**

1. **Set default during deployment:**
   ```bash
   aws cloudformation update-stack \
     --stack-name my-stack \
     --parameters ParameterKey=DefaultSenderEmail,ParameterValue=noreply@yourcompany.com
   ```

2. **Add to template metadata.json:**
   ```json
   {
     "senderEmail": "noreply@yourcompany.com",
     "senderName": "Your Company"
   }
   ```

3. **Override per email:**
   ```json
   {
     "detail": {
       "senderOverride": {
         "email": "noreply@yourcompany.com",
         "name": "Your Company"
       }
     }
   }
   ```

**Priority order:** senderOverride > template metadata > DefaultSenderEmail

---

## Suppression Errors

### Email not sent: recipient is suppressed

**Error:** `Email not sent: recipient is suppressed (hard-bounce)`

**When:** The recipient email is on your suppression list due to previous bounces or complaints.

**Reasons:**
- `hard-bounce` - Email address doesn't exist
- `soft-bounce-exceeded` - Too many temporary failures
- `spam-complaint` - Recipient marked your email as spam

**Fix:**
1. Check why the email is suppressed:
   ```bash
   aws dynamodb get-item \
     --table-name ${STACK_NAME}-Suppression \
     --key '{"email": {"S": "user@example.com"}}'
   ```

2. If the address is now valid, remove from suppression:
   ```bash
   aws dynamodb delete-item \
     --table-name ${STACK_NAME}-Suppression \
     --key '{"email": {"S": "user@example.com"}}'
   ```

**Warning:** Only remove addresses you've verified are now valid. Sending to invalid addresses damages your SES reputation.

---

### Manual Suppression Removal (Best Practice)

When a user contacts support saying "I accidentally marked your email as spam" or "My email address is working now", follow this process:

**Why manual removal is required:**
- Most email providers (Gmail, Yahoo, Outlook) do NOT send "not-spam" feedback when users un-mark emails as spam
- Auto-removing based on unreliable "not-spam" events is risky
- Manual verification ensures you only send to users who genuinely want your emails

**Process:**
1. **User complains** → System auto-suppresses (this is correct behavior)
2. **User contacts support** → "I accidentally marked as spam, please re-add me"
3. **Admin verifies** → Confirm the request is legitimate
4. **Admin removes from suppression:**
   ```bash
   aws dynamodb delete-item \
     --table-name ${STACK_NAME}-Suppression \
     --key '{"email": {"S": "user@example.com"}}'
   ```
5. **User can receive emails again**

**Before removing, verify:**
- The request came from the actual email owner (not spoofed)
- For bounces: the email address is now valid (user fixed their mailbox)
- For complaints: the user genuinely wants to receive emails again

**Do NOT auto-remove** based on:
- SES "not-spam" feedback (unreliable, rarely sent by providers)
- User clicking links in old emails (doesn't indicate consent)
- Time elapsed since suppression (addresses don't become valid over time)

---

## Bounce Rate Errors

### Email not sent: bounce rate exceeded

**Error:** `Email not sent: bounce rate exceeded`

**When:** Your daily bounce rate exceeds the configured threshold (default 5%).

**Fix:**
1. Check current bounce rate in CloudWatch Logs
2. Wait for the rate to recover (calculated daily)
3. Clean your email list to remove invalid addresses
4. Temporarily increase threshold if needed:
   ```bash
   aws cloudformation update-stack \
     --stack-name my-stack \
     --parameters ParameterKey=BounceRateThreshold,ParameterValue=7
   ```

**Prevention:**
- Use double opt-in for email collection
- Regularly clean your email lists
- Monitor bounce rates proactively

See [BOUNCE_RATE_PROTECTION.md](BOUNCE_RATE_PROTECTION.md) for details.

---

## SES Errors

### MessageRejected

**Error:** `SES error (MessageRejected): Email address is not verified`

**When:** The sender email or domain isn't verified in SES.

**Fix:**
1. Verify your domain or email in SES console
2. Check verification status:
   ```bash
   aws ses get-identity-verification-attributes \
     --identities noreply@yourcompany.com
   ```

---

### MailFromDomainNotVerified

**Error:** `SES error (MailFromDomainNotVerified): ...`

**When:** Custom MAIL FROM domain isn't verified.

**Fix:** Verify your MAIL FROM domain in SES console or use the default SES domain.

---

### ConfigurationSetDoesNotExist

**Error:** `SES error (ConfigurationSetDoesNotExist): ...`

**When:** The SES configuration set was deleted or doesn't exist.

**Fix:** Check that the stack deployed correctly:
```bash
aws ses describe-configuration-set \
  --configuration-set-name ${STACK_NAME}-ConfigSet
```

---

### AccountSendingPaused

**Error:** `SES error (AccountSendingPaused): ...`

**When:** AWS has paused your SES sending due to reputation issues.

**Fix:**
1. Check SES console for account status
2. Review bounce and complaint rates
3. Contact AWS Support if needed

---

### Throttling / TooManyRequestsException

**Error:** `SES error (Throttling): Rate exceeded` or `Maximum sending rate exceeded`

**When:** You're sending emails faster than your SES quota allows.

**What happens:**
1. Lambda retries internally (2 attempts with exponential backoff)
2. If still failing, Lambda throws exception
3. EventBridge retries (3 attempts with backoff)
4. If all retries fail, message goes to EventBridge DLQ

**Important:** Throttled emails that exhaust all retries end up in the DLQ. These messages are safely stored and can be reprocessed once your sending rate normalizes. The DLQ alarm will notify you when messages require attention. See "DLQ Depth Alarms" section below for reprocessing instructions.

---

### Throttling During Burst Loads (Expected Behavior)

**Symptom:** CloudWatch shows Lambda errors during high-volume sends, but all emails eventually deliver.

**Example:** You send 10,000 emails in 20 seconds. CloudWatch shows 100+ errors in the first minute, but final delivery count is 10,000/10,000.

**What's happening:**
- Your application submits events faster than SES can process them
- Initial burst causes throttling errors (e.g., "Maximum sending rate exceeded")
- Lambda's retry logic and EventBridge's retry policy spread out the load
- System self-corrects and all emails eventually deliver

**This is expected behavior, not a bug.** The error handling and retry mechanisms are working correctly.

**Timeline example (10,000 emails, SES rate 14/sec):**
- 0-20 seconds: Events submitted (493 events/sec)
- 0-60 seconds: Initial throttling errors as burst exceeds SES capacity
- 1-15 minutes: System processes backlog at SES rate limit
- Final result: 100% delivery, 0% data loss

**CloudWatch Alarm Emails During Burst Loads:**

You may receive alarm notification emails like:
```
Alarm "EmailSender-Errors" has entered the ALARM state
Reason: Threshold Crossed: 2 datapoints [75.0, 60.0] were greater than the threshold (50.0)
```

**Note:** The alarm is configured to only trigger when errors exceed 50 for 2 consecutive 5-minute periods (10 minutes total). This means:
- Temporary burst throttling (like your load test) won't trigger alarms
- Only sustained errors over 10+ minutes will alert you
- If you DO receive an alarm, it indicates a real problem that needs attention

**When to be concerned:**
- Alarm triggers (sustained errors for 10+ minutes)
- Alarm doesn't transition back to OK after investigation
- Emails NOT delivering after retries (check DLQ)
- DLQ depth alarm triggered

**When NOT to be concerned:**
- Temporary errors during burst loads (alarm won't trigger)
- All emails eventually delivered
- DLQ remains empty

**Tuning alarm sensitivity (optional):**

The default alarm configuration is balanced for most production workloads:
- Threshold: >50 errors
- Evaluation: 2 consecutive 5-minute periods

For stricter monitoring (low-volume, critical emails):
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name ${STACK_NAME}-EmailSender-Errors \
  --threshold 10 \
  --evaluation-periods 1 \
  --comparison-operator GreaterThanThreshold \
  --metric-name Errors \
  --namespace AWS/Lambda \
  --dimensions Name=FunctionName,Value=${STACK_NAME}-EmailSender \
  --period 300 \
  --statistic Sum \
  --alarm-actions arn:aws:sns:${REGION}:${ACCOUNT}:${STACK_NAME}-Alarms
```

---

### Fix - Match Concurrency to Your SES Rate

1. Check your SES sending rate:
   ```bash
   aws ses get-send-quota --region YOUR_REGION
   ```
   Look for `MaxSendRate` (e.g., 14.0 means 14 emails/second)

2. Update your stack to match:
   ```bash
   aws cloudformation update-stack \
     --stack-name my-email-engine \
     --use-previous-template \
     --parameters \
       ParameterKey=EmailSenderConcurrency,ParameterValue=14 \
     --capabilities CAPABILITY_NAMED_IAM
   ```

**How it works:** The `EmailSenderConcurrency` parameter limits how many Lambda instances can run simultaneously. If you set it to 14 and your SES rate is 14/sec, events queue up in EventBridge instead of overwhelming SES.

| SES MaxSendRate | Recommended EmailSenderConcurrency |
|-----------------|-----------------------------------|
| 1 (sandbox) | 1 |
| 14 (typical production) | 14 (default) |
| 50+ (scaled) | Match your MaxSendRate |

**For persistent issues:**
1. Request a sending quota increase in SES console
2. Implement rate limiting in your application
3. Monitor the EventBridge DLQ alarm for throttling-related failures
4. Consider spreading sends over time instead of bursts

**Checking for throttled emails in DLQ:**
```bash
# Get DLQ URL
DLQ_URL=$(aws cloudformation describe-stacks \
  --stack-name my-email-engine \
  --query 'Stacks[0].Outputs[?OutputKey==`EventBridgeDLQUrl`].OutputValue' \
  --output text)

# Check for messages
aws sqs receive-message --queue-url $DLQ_URL --max-number-of-messages 10
```

---

## Validation Errors

### Missing required field: 'to'

**Error:** `Validation error: Missing required field: 'to'`

**When:** Your EventBridge event is missing the recipient email.

**Fix:** Include the `to` field:
```json
{
  "detail": {
    "to": "recipient@example.com",
    "templateName": "welcome"
  }
}
```

---

### Missing required field: 'templateName'

**Error:** `Validation error: Missing required field: 'templateName'`

**When:** Your EventBridge event is missing the template name.

**Fix:** Include the `templateName` field:
```json
{
  "detail": {
    "to": "recipient@example.com",
    "templateName": "welcome"
  }
}
```

---

### Invalid email address format

**Error:** `Validation error: Invalid email address format: not-an-email`

**When:** The `to` field contains an invalid email address.

**Fix:** Provide a valid email address format:
```json
{
  "detail": {
    "to": "valid@example.com"
  }
}
```

---

### No subject provided

**Error:** `Validation error: No subject provided in event or template`

**When:** Neither the event nor the template metadata includes a subject.

**Fix (choose one):**
1. Add subject to your event:
   ```json
   {
     "detail": {
       "subject": "Welcome to our service"
     }
   }
   ```

2. Add subject to template metadata.json:
   ```json
   {
     "subject": "Welcome to {{ companyName }}!"
   }
   ```

---

## AWS Service Errors

### DynamoDB Errors

**Error:** `DynamoDB error checking suppression: ProvisionedThroughputExceededException`

**What happens:** The system will automatically retry.

**If persistent:** Your tables use on-demand billing, so this shouldn't happen. Check CloudWatch for unusual traffic patterns.

---

### S3 Errors

**Error:** `S3 error loading template: AccessDenied`

**When:** Lambda doesn't have permission to read from the template bucket.

**Fix:** This indicates a deployment issue. Redeploy the stack or check IAM permissions.

---

## Checking Logs

### Find errors in CloudWatch

```bash
# Get log group name
LOG_GROUP="/aws/lambda/${STACK_NAME}-EmailSender"

# Search for errors
aws logs filter-log-events \
  --log-group-name $LOG_GROUP \
  --filter-pattern "ERROR" \
  --start-time $(date -d '1 hour ago' +%s000)
```

### Common log patterns

| Pattern | What it shows |
|---------|---------------|
| `"ERROR"` | All errors |
| `"Template error"` | Template loading/rendering issues |
| `"SES error"` | Email sending failures |
| `"suppressed"` | Blocked due to suppression |
| `"bounce_rate_exceeded"` | Blocked due to bounce rate |

---

## Feedback Processor Errors

These errors occur when processing bounce, complaint, and delivery notifications from SES.

### Failed to parse SNS message

**Error:** `Failed to parse SNS message`

**When:** The Feedback Processor Lambda received an invalid SNS notification.

**What happens:** The message goes to the Feedback Processor DLQ for investigation.

**Fix:** This is usually an AWS infrastructure issue. Check:
1. SNS topic configuration is correct
2. SES configuration set event destinations are properly configured

---

### Unknown notification type

**Error:** `Unknown notification type: X`

**When:** SES sent a notification type the system doesn't recognize.

**What happens:** The notification is logged and discarded.

**Fix:** This is informational. The system handles Bounce, Complaint, Delivery, and Open notifications. Other types are safely ignored.

---

### Email record not found for SES message ID

**Error:** `Email record not found for SES message ID: X`

**When:** A bounce/complaint/delivery notification arrived but no matching email tracking record exists.

**Possible causes:**
- Email was sent before SESMailEngine was deployed
- Tracking record was deleted (TTL expired)
- Race condition (notification arrived before tracking record was written)

**What happens:** The notification is processed for suppression (if bounce/complaint) but tracking update is skipped.

**Fix:** Usually no action needed. If persistent, check that Email Sender Lambda is creating tracking records.

---

## Template Seeder Errors

### Template Seeder Lambda not found

**Error:** `ResourceNotFoundException: Function not found: my-email-engine-TemplateSeeder`

**When:** The Template Seeder Lambda doesn't exist.

**Fix:**
1. Verify the stack deployed correctly:
   ```bash
   aws cloudformation describe-stacks --stack-name my-email-engine
   ```
2. Check if the Lambda exists:
   ```bash
   aws lambda get-function --function-name my-email-engine-TemplateSeeder
   ```
3. If missing, the stack may need to be updated or redeployed

---

### No starter templates found

**Response:** `{"installed": [], "skipped": [], "message": "No starter templates found in source bucket"}`

**When:** The Marketplace artifacts bucket doesn't contain starter templates.

**Fix:** This is expected if the Marketplace bucket hasn't been populated yet. For development:
1. Upload templates to the source bucket manually
2. Or create templates directly in your template bucket

---

### AccessDenied on source bucket

**Error:** `AccessDenied: Access Denied` when reading from `sesmailengine-artifacts`

**When:** The Lambda doesn't have permission to read from the Marketplace bucket.

**Fix:**
1. Verify the IAM role has S3 read permissions:
   ```bash
   aws iam get-role-policy \
     --role-name my-email-engine-TemplateSeederRole \
     --policy-name my-email-engine-TemplateSeederPolicy
   ```
2. Check the bucket policy allows public read access

---

### AccessDenied on destination bucket

**Error:** `AccessDenied: Access Denied` when writing to template bucket

**When:** The Lambda doesn't have permission to write to the customer's template bucket.

**Fix:**
1. Verify the IAM role has S3 write permissions
2. Check the bucket policy doesn't block the Lambda role

---

### Templates not appearing after seeder runs

**Symptom:** Seeder reports success but templates aren't in S3.

**Fix:**
1. Check the seeder response for actual installed templates:
   ```bash
   aws lambda invoke --function-name my-email-engine-TemplateSeeder /dev/stdout
   ```
2. Verify templates in S3:
   ```bash
   aws s3 ls s3://$(aws cloudformation describe-stacks \
     --stack-name my-email-engine \
     --query 'Stacks[0].Outputs[?OutputKey==`TemplateBucketName`].OutputValue' \
     --output text)/templates/
   ```
3. If `skipped` contains templates, they already exist (seeder won't overwrite)

---

## Getting Help

If you can't resolve an issue:

1. Check CloudWatch Logs for detailed error messages
2. Verify your EventBridge event format matches the schema
3. Ensure all required AWS resources exist (S3 bucket, DynamoDB tables)
4. Check SES console for account-level issues

For template issues, test locally first:
```python
from jinja2 import Template
t = Template(open('template.html').read())
print(t.render(userName="Test", companyName="Acme"))
```

---

## CloudWatch Alarm Troubleshooting

### DLQ Depth Alarms

**Alarm:** `EventBridge DLQ Depth`, `Retry DLQ Depth`, or `Feedback DLQ Depth`

**When:** Messages are accumulating in a dead letter queue.

**Important:** Messages in the DLQ are NOT lost - they are safely stored and can be reprocessed. The DLQ alarm notifies you that manual attention is required.

**Common causes for EventBridge DLQ messages:**
- SES throttling during high-volume sends (see "Throttling During Burst Loads" above)
- Template errors that persist across retries
- Transient AWS service issues

**Fix:**
1. Check the DLQ for messages:
   ```bash
   aws sqs receive-message \
     --queue-url $(aws cloudformation describe-stacks \
       --stack-name my-email-engine \
       --query 'Stacks[0].Outputs[?OutputKey==`EventBridgeDLQUrl`].OutputValue' \
       --output text) \
     --max-number-of-messages 10
   ```
2. Review message content to identify the failure cause
3. Fix the underlying issue (template errors, permissions, SES rate limits, etc.)
4. Reprocess messages by re-publishing to EventBridge, or delete if no longer needed

**Reprocessing DLQ messages:**
```bash
# Read message from DLQ
MESSAGE=$(aws sqs receive-message --queue-url $DLQ_URL --max-number-of-messages 1)

# Extract the original event payload
PAYLOAD=$(echo $MESSAGE | jq -r '.Messages[0].Body' | jq '.requestPayload')

# Re-publish to EventBridge
aws events put-events --entries "[{
  \"Source\": \"reprocess.dlq\",
  \"DetailType\": \"Email Request\",
  \"Detail\": $(echo $PAYLOAD | jq -c '.detail'),
  \"EventBusName\": \"${STACK_NAME}-EmailBus\"
}]"

# Delete from DLQ after successful reprocessing
aws sqs delete-message --queue-url $DLQ_URL \
  --receipt-handle $(echo $MESSAGE | jq -r '.Messages[0].ReceiptHandle')
```

**Note:** For SES throttling-related DLQ messages, wait until your sending rate normalizes before reprocessing to avoid immediate re-throttling.

---

### Lambda Error Alarms

**Alarm:** `Email Sender Errors` or `Feedback Processor Errors`

**When:** Lambda functions are experiencing sustained errors (>50 errors for 2 consecutive 5-minute periods).

**Why this threshold:** The alarm is configured to ignore temporary burst throttling while catching real problems. A single spike during a marketing campaign won't trigger an alert, but sustained errors over 10+ minutes will.

**Fix:**
1. Check CloudWatch Logs for error details:
   ```bash
   aws logs filter-log-events \
     --log-group-name /aws/lambda/${STACK_NAME}-EmailSender \
     --filter-pattern "ERROR" \
     --start-time $(date -d '1 hour ago' +%s000)
   ```
2. Common causes:
   - Template not found or invalid
   - DynamoDB throttling
   - SES sending limits exceeded
   - IAM permission issues

---

### SES Bounce Rate Alarm

**Alarm:** `SES Bounce Rate > 3%`

**When:** Your bounce rate exceeds 3% (AWS suspends at ~5%).

**Fix:**
1. Check your bounce rate in SES console
2. Review recent bounces in the Suppression table:
   ```bash
   aws dynamodb scan \
     --table-name ${STACK_NAME}-Suppression \
     --filter-expression "suppressionType = :type" \
     --expression-attribute-values '{":type": {"S": "bounce"}}'
   ```
3. Clean your email list to remove invalid addresses
4. Implement double opt-in for new subscribers
5. See [BOUNCE_RATE_PROTECTION.md](BOUNCE_RATE_PROTECTION.md) for details

---

### SES Complaint Rate Alarm

**Alarm:** `SES Complaint Rate > 0.05%`

**When:** Your complaint rate exceeds 0.05% (AWS suspends at ~0.1%).

**Fix:**
1. Check your complaint rate in SES console
2. Review recent complaints in the Suppression table:
   ```bash
   aws dynamodb scan \
     --table-name ${STACK_NAME}-Suppression \
     --filter-expression "suppressionType = :type" \
     --expression-attribute-values '{":type": {"S": "complaint"}}'
   ```
3. Review your email content and frequency
4. Ensure clear unsubscribe options in all emails
5. Only send to users who explicitly opted in
