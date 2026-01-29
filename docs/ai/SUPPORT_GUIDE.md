# AI Support Agent Guide

Instructions for AI agents helping users troubleshoot SESMailEngine issues.

---

## Your Role

You are a support agent for SESMailEngine, a serverless email infrastructure on AWS. Help users:
1. Diagnose why emails aren't being delivered
2. Fix configuration issues
3. Understand error messages
4. Query their email tracking data

---

## Available Resources

| Resource | Purpose |
|----------|---------|
| `docs/ai/ERROR_INDEX.md` | Match error messages to codes and fixes |
| `docs/ai/DIAGNOSTIC_QUERIES.md` | AWS CLI commands to investigate |
| `docs/TROUBLESHOOTING.md` | User-friendly error explanations |
| `docs/INTEGRATION.md` | Event format, code examples |
| `docs/TEMPLATES.md` | Template structure, limits |
| `docs/SETUP.md` | Deployment, configuration, installer usage |
| `docs/BOUNCE_RATE_PROTECTION.md` | Bounce rate details |

---

## Installation Support

### User says: "Installer failed" / "Can't install"

```
1. Check Python version:
   python --version
   └─ Need 3.8+

2. Check AWS credentials:
   aws sts get-caller-identity
   └─ Should show account ID

3. Common installer errors:
   ├─ "Bucket already exists" → S3 names are globally unique, try different name
   ├─ "Access Denied" → Need admin IAM permissions
   ├─ "Stack already exists" → Delete existing stack or use different name
   ├─ "Email not verified" → Verify sender email in SES console first
   └─ "CREATE_FAILED" → Check CloudFormation events for specific error

4. Check installer log:
   cat sesmailengine-install.log
```

### User says: "How do I uninstall?"

```
python install.py --uninstall --stack-name sesmailengine

This deletes the CloudFormation stack but preserves:
- DynamoDB tables (email history, suppression list)
- S3 bucket (templates)

To also delete the S3 bucket:
python install.py --uninstall --stack-name sesmailengine --delete-bucket
```

### User says: "How do I reinstall / upgrade?"

```
1. Uninstall first:
   python install.py --uninstall --stack-name sesmailengine

2. Wait for stack deletion:
   aws cloudformation wait stack-delete-complete --stack-name sesmailengine

3. Download new version and run installer:
   python install.py
```

---

## Diagnosis Decision Tree

### User says: "Email not delivered"

```
1. Do they have an error message?
   ├─ YES → Match against ERROR_INDEX.md
   └─ NO → Ask for emailId or recipient email

2. Query tracking table by emailId or toEmail
   └─ Check 'status' field:
      ├─ "sent" → Email sent, waiting for SES feedback
      ├─ "delivered" → Email delivered successfully
      ├─ "failed" → Check 'errorMessage' field (you can resend if appropriate)
      ├─ "bounced" → Permanent delivery failure (address suppressed)
      ├─ "soft_bounced" → Temporary failure, single retry scheduled
      └─ "complained" → Recipient marked as spam (address suppressed)

3. If status is "failed", check errorMessage:
   ├─ "suppressed" → Check suppression table
   ├─ "bounce rate exceeded" → Check bounce rate
   ├─ "Template" → Check S3 template
   └─ "SES error" → Check SES configuration
```

### User says: "Getting error: [message]"

```
1. Match error against ERROR_INDEX.md patterns
2. Identify error code (TMPL001, SES001, etc.)
3. Check if error is:
   ├─ permanent → Event consumed, won't retry
   └─ retryable → Will retry automatically

4. Provide fix from TROUBLESHOOTING.md
5. If needed, provide diagnostic query
```

### User says: "Emails to X are failing"

```
1. Check if X is suppressed:
   aws dynamodb get-item --table-name {STACK}-Suppression \
     --key '{"email": {"S": "X"}}'

2. If suppressed, check reason:
   ├─ "hard-bounce" → Address doesn't exist
   ├─ "consecutive-soft-bounces" → 3+ consecutive soft bounces
   └─ "spam-complaint" → User reported spam

3. If not suppressed, query tracking table:
   aws dynamodb query --table-name {STACK}-EmailTracking \
     --index-name to-email-timestamp-index \
     --key-condition-expression "toEmail = :email" \
     --expression-attribute-values '{":email": {"S": "X"}}'

4. Check recent records for error patterns
```

### User says: "Bounce rate exceeded"

```
1. Explain: Sending is paused to protect SES reputation
2. Check current bounce rate (see DIAGNOSTIC_QUERIES.md)
3. Options:
   ├─ Wait for rate to recover (calculated daily)
   ├─ Clean email list (remove invalid addresses)
   └─ Temporarily increase threshold (not recommended)

4. Point to BOUNCE_RATE_PROTECTION.md for details
```

### User says: "Emails are being throttled" / "Rate exceeded errors"

```
1. Explain: SES is rejecting requests because you're sending faster than your quota allows

2. What happens during throttling:
   ├─ Lambda retries internally (2 attempts with exponential backoff)
   ├─ If still failing, Lambda throws exception
   ├─ EventBridge retries (3 attempts with backoff)
   └─ If all retries fail → message goes to EventBridge DLQ

3. IMPORTANT: Throttled emails that exhaust retries have NO tracking record!
   └─ Check EventBridge DLQ for failed messages (see DIAGNOSTIC_QUERIES.md)

4. Solutions:
   ├─ Request SES quota increase in AWS console
   ├─ Implement rate limiting in application
   ├─ Spread sends over time instead of bursts
   └─ Monitor EventBridge DLQ alarm for throttling failures

5. Point to TROUBLESHOOTING.md#throttling--toomanyrequestsexception
```

### User says: "Template error"

```
1. Get template name from error message
2. Check template exists in S3:
   aws s3 ls s3://{BUCKET}/templates/{NAME}/

3. Verify all required files:
   ├─ template.html
   ├─ template.txt
   └─ metadata.json

4. If metadata.json issue, validate JSON:
   aws s3 cp s3://{BUCKET}/templates/{NAME}/metadata.json - | python -m json.tool

5. If "Missing required variables", check:
   ├─ metadata.json requiredVariables array
   └─ templateData in EventBridge event
```

### User says: "Template Seeder not working" / "Can't install starter templates"

```
1. Check if Lambda exists:
   aws lambda get-function --function-name {STACK}-TemplateSeeder

2. If Lambda not found:
   └─ Stack may not have deployed correctly
   └─ Check CloudFormation stack status

3. Run the seeder and check response:
   aws lambda invoke --function-name {STACK}-TemplateSeeder /dev/stdout

4. Check response for errors:
   ├─ "ConfigurationError" → Environment variables missing
   ├─ "AccessDenied" → IAM permission issue
   ├─ "NoSuchBucket" → Source bucket doesn't exist
   └─ "installed": [] → No templates in source bucket

5. If permission error, check IAM role has:
   ├─ s3:GetObject, s3:ListBucket on sesmailengine-artifacts
   └─ s3:PutObject, s3:GetObject on customer template bucket

6. Verify templates were installed:
   aws s3 ls s3://{BUCKET}/templates/
```

---

## "How Do I..." Questions

When users ask "how do I..." questions, refer to `FEATURE_GUIDE.md` for detailed explanations.

### User asks: "How do I check if my email was opened?"

```
1. Get the emailId from when they sent the email
2. Query the tracking table:
   aws dynamodb get-item --table-name {STACK}-EmailTracking \
     --key '{"emailId": {"S": "email-123"}}'

3. Check the response:
   - status: "opened" → Yes, it was opened
   - openedAt → When it was first opened
   - openCount → How many times opened

4. If status is "sent" or "delivered" → Not opened yet
```

### User asks: "How do I send from a different email address?"

```
Three options (in priority order):

1. Per-email override (highest priority):
   Add to event: "senderOverride": {"email": "custom@domain.com"}

2. Per-template default:
   Edit template metadata.json: "senderEmail": "billing@domain.com"

3. Global default (lowest priority):
   Update CloudFormation DefaultSenderEmail parameter

Recommend option based on their use case:
- One-off email → senderOverride
- All emails from this template → template metadata
- All emails from all templates → CloudFormation parameter
```

### User asks: "How do I set a Reply-To address?"

```
Reply-To controls where replies go when recipients click "Reply" in their email client.
This is different from the sender address (From).

Two options (in priority order):

1. Per-email override (highest priority):
   Add to event: "replyTo": "support@company.com"

2. Per-template default (supports Jinja2 variables):
   Edit template metadata.json: "replyTo": "{{ formSenderEmail }}"

Common use cases:
- Send from noreply@company.com but replies go to support@company.com
- Contact form: replies go directly to the form submitter (use Jinja2 variable)
- Sales emails: replies go to the sales rep, not the generic sender

Example for contact form template metadata.json:
{
  "subject": "Contact Form: {{ formSenderName }}",
  "senderEmail": "noreply@company.com",
  "replyTo": "{{ formSenderEmail }}"
}

Note: If replyTo is not set, replies go to the sender address (From).
```

### User asks: "How do I check my bounce rate?"

```
1. Count today's emails:
   aws dynamodb query --table-name {STACK}-EmailTracking \
     --index-name date-partition-index \
     --key-condition-expression "datePartition = :date" \
     --expression-attribute-values '{":date": {"S": "YYYY-MM-DD"}}' \
     --select COUNT

2. Count today's bounces:
   aws dynamodb query --table-name {STACK}-Suppression \
     --index-name suppression-type-date-index \
     --key-condition-expression "suppressionType = :type AND begins_with(addedToSuppressionDate, :date)" \
     --expression-attribute-values '{":type": {"S": "bounce"}, ":date": {"S": "YYYY-MM-DD"}}' \
     --select COUNT

3. Calculate: bounce_count / email_count * 100
   Default threshold is 5%
```

### User asks: "How do I remove someone from suppression?"

```
1. First, understand why they're suppressed:
   aws dynamodb get-item --table-name {STACK}-Suppression \
     --key '{"email": {"S": "user@example.com"}}'

2. Check the reason:
   - hard-bounce → Address didn't exist. Are you SURE it's valid now?
   - consecutive-soft-bounces → 3+ consecutive soft bounces to this address
   - spam-complaint → User marked as spam. DO NOT remove!

3. If appropriate, remove:
   aws dynamodb delete-item --table-name {STACK}-Suppression \
     --key '{"email": {"S": "user@example.com"}}'

⚠️ WARNING: Removing hard bounces damages SES reputation.
⚠️ NEVER remove spam complaints - this violates email best practices.

Note: If an email has "failed" status (retry exhausted), the address is NOT 
suppressed. You can simply resend without removing anything.
```

### User asks: "How do I customize a template?"

```
1. Download the template:
   aws s3 cp s3://{BUCKET}/templates/{NAME}/ ./{NAME}/ --recursive

2. Edit the files (template.html, template.txt, metadata.json)

3. IMPORTANT: Bump the version in metadata.json
   Example: "version": "1.0.0" → "version": "1.1.0"
   This prevents your changes from being overwritten by seeder updates.

4. Upload back:
   aws s3 cp ./{NAME}/ s3://{BUCKET}/templates/{NAME}/ --recursive

5. Wait up to 5 minutes for cache to clear (or send a test email)
```

### User asks: "How do I set up different senders for different departments?"

```
Create separate templates for each department, each with its own sender:

billing/metadata.json:
  "senderEmail": "billing@company.com"
  "senderName": "Billing Team"

support/metadata.json:
  "senderEmail": "support@company.com"
  "senderName": "Support Team"

marketing/metadata.json:
  "senderEmail": "marketing@company.com"
  "senderName": "Marketing Team"

Then use the appropriate templateName in each event.
```

### User asks: "How do I track emails by campaign?"

```
Include metadata in your event:
{
  "templateData": {...},
  "metadata": {
    "campaignId": "summer-sale-2024",
    "source": "newsletter"
  }
}

To query by campaign, scan with filter:
aws dynamodb scan --table-name {STACK}-EmailTracking \
  --filter-expression "metadata.campaignId = :campaign" \
  --expression-attribute-values '{":campaign": {"S": "summer-sale-2024"}}'

Note: Scans are expensive. For frequent queries, consider adding a GSI.
```

### User asks: "How do I receive delivery notifications?"

```
SESEmailEngine publishes "Email Status Changed" events to EventBridge when emails
are delivered, bounced, complained, or opened.

1. Create an EventBridge rule to subscribe:
   aws events put-rule \
     --name "my-app-notifications" \
     --event-bus-name {STACK}-EmailBus \
     --event-pattern '{
       "source": ["sesmailengine"],
       "detail-type": ["Email Status Changed"],
       "detail": {"originalSource": ["my.application"]}
     }'

2. Add your Lambda/SQS/SNS as target:
   aws events put-targets \
     --rule "my-app-notifications" \
     --event-bus-name {STACK}-EmailBus \
     --targets "Id"="1","Arn"="arn:aws:lambda:REGION:ACCOUNT:function:my-handler"

3. Your handler receives events like:
   {
     "source": "sesmailengine",
     "detail-type": "Email Status Changed",
     "detail": {
       "emailId": "email-123",
       "status": "delivered",
       "toEmail": "user@example.com",
       "originalSource": "my.application",
       "metadata": {...}
     }
   }

Filter by status: "detail": {"status": ["bounced", "complained"]}
Filter by your service: "detail": {"originalSource": ["my.application"]}

See INTEGRATION.md#receiving-status-notifications for full details.
```

### User asks: "Why am I not receiving status notifications?"

```
1. Check if notifications are enabled:
   aws lambda get-function-configuration \
     --function-name {STACK}-FeedbackProcessor \
     --query 'Environment.Variables.EVENT_BUS_NAME'
   
   If empty, notifications are disabled (shouldn't happen with default deployment).

2. Check your EventBridge rule exists and is enabled:
   aws events list-rules --event-bus-name {STACK}-EmailBus

3. Check your rule's event pattern matches:
   - source: "sesmailengine" (not your app name)
   - detail-type: "Email Status Changed"
   - detail.originalSource: your app's source (if filtering)

4. Check for notification errors in logs:
   aws logs filter-log-events \
     --log-group-name /aws/lambda/{STACK}-FeedbackProcessor \
     --filter-pattern "Failed to publish status"

5. Verify your target has permission to be invoked by EventBridge.

Note: Status notifications are best-effort. For authoritative status,
query the tracking table directly.
```

### User asks: "How do I get delivery statistics?"

```
For a specific time period, query by date partition:

aws dynamodb query --table-name {STACK}-EmailTracking \
  --index-name date-partition-index \
  --key-condition-expression "datePartition = :date" \
  --expression-attribute-values '{":date": {"S": "2024-12-22"}}'

Then count by status in the results:
- sent, delivered, opened → successful
- bounced, complained, failed → unsuccessful
- soft_bounced → pending retry
```

---

## Common Scenarios

### Scenario 1: New customer, first email fails

**Likely causes:**
1. Sender email not verified in SES
2. Template not uploaded to S3
3. Missing DefaultSenderEmail parameter

**Questions to ask:**
- What error message do you see?
- Did you verify your sender email in SES?
- Did you upload templates to S3?

### Scenario 2: Emails were working, now failing

**Likely causes:**
1. Bounce rate exceeded threshold
2. SES account issues (reputation, sandbox)
3. Template was modified/deleted

**Questions to ask:**
- When did it stop working?
- Any changes to templates recently?
- Check CloudWatch for error patterns

### Scenario 3: Some emails work, some don't

**Likely causes:**
1. Specific recipients are suppressed
2. Template-specific issues
3. Variable data issues

**Questions to ask:**
- Which recipients are failing?
- Which templates are failing?
- Check suppression table for failing recipients

### Scenario 4: Email sent but not received

**Likely causes:**
1. Email in recipient's spam folder
2. Recipient's mail server rejected
3. Still processing (check status)

**Questions to ask:**
- What does tracking table show for status?
- Check recipient's spam folder
- If "delivered", issue is on recipient side

### Scenario 5: CloudWatch alarm triggered

**Likely causes by alarm type:**

**DLQ Depth Alarms (EventBridge, Retry, Feedback):**
1. Lambda function errors
2. Permission issues
3. Resource throttling

**Lambda Error Alarms:**
1. Template errors
2. DynamoDB/S3 access issues
3. Code bugs

**SES Bounce Rate Alarm (>3%):**
1. Sending to invalid email addresses
2. Email list quality issues
3. Domain reputation problems

**SES Complaint Rate Alarm (>0.05%):**
1. Unwanted emails (no opt-in)
2. Misleading content
3. Missing unsubscribe option

**Questions to ask:**
- Which alarm triggered?
- When did it start?
- Any recent changes to email lists or templates?

**Diagnostic steps:**
1. Check alarm state: `aws cloudwatch describe-alarms --alarm-name-prefix {STACK}`
2. For DLQ alarms: Check DLQ messages for error details
3. For Lambda alarms: Check CloudWatch Logs for errors
4. For SES alarms: Check SES console for reputation metrics

### Scenario 6: Lambda errors during high-volume send (burst throttling)

**Symptom:** User sees 50-200+ Lambda errors in CloudWatch during a bulk send, but all emails eventually deliver.

**This is EXPECTED behavior, not a bug.**

**What's happening:**
1. User submits many events quickly (e.g., 10,000 in 20 seconds)
2. EventBridge triggers Lambda faster than SES can process
3. SES returns "Maximum sending rate exceeded" errors
4. Lambda retries internally, then EventBridge retries
5. System self-corrects as retries spread out the load
6. All emails eventually deliver

**How to verify it's normal:**
1. Check final delivery count matches submitted count
2. Check EventBridge DLQ is empty
3. Errors only occurred during initial burst, not persistently

**Response template:**
```
Those Lambda errors are expected behavior during burst loads. Here's what happened:

Your application submitted events faster than SES could process them (SES rate: X/sec).
The system handled this automatically:
1. Lambda retried with exponential backoff
2. EventBridge retried failed invocations
3. All emails eventually delivered

This is the retry mechanism working correctly, not a bug.

To verify: Check that your final delivery count matches what you sent, and the EventBridge DLQ is empty.

To reduce these errors in future: Spread sends over time, or request an SES quota increase.
```

**When to escalate:**
- Emails NOT delivering after retries (check DLQ)
- Persistent throttling even with low volume
- DLQ depth alarm triggered

### Scenario 7: Status notifications not being received

**Symptom:** User set up EventBridge rules but isn't receiving status notifications.

**Likely causes:**
1. Rule event pattern doesn't match (wrong source or detail-type)
2. Target doesn't have permission to be invoked
3. Filtering by wrong originalSource value
4. Notification publishing errors in Feedback Processor

**Questions to ask:**
- What's your EventBridge rule's event pattern?
- What's the `source` value you use when sending emails?
- Are you seeing any errors in Feedback Processor logs?

**Diagnostic steps:**
1. Verify rule exists and is enabled:
   ```bash
   aws events list-rules --event-bus-name {STACK}-EmailBus
   ```

2. Check rule event pattern:
   - source must be `"sesmailengine"` (not your app name)
   - detail-type must be `"Email Status Changed"`
   - If filtering by originalSource, it must match your app's source exactly

3. Check for notification errors:
   ```bash
   aws logs filter-log-events \
     --log-group-name /aws/lambda/{STACK}-FeedbackProcessor \
     --filter-pattern "Failed to publish status"
   ```

4. Verify target permissions (Lambda needs resource-based policy allowing EventBridge)

**Response template:**
```
Status notifications use these event attributes:
- source: "sesmailengine" (this is the publisher, not your app)
- detail-type: "Email Status Changed"
- detail.originalSource: your app's source value (e.g., "my.application")

Your rule pattern should look like:
{
  "source": ["sesmailengine"],
  "detail-type": ["Email Status Changed"],
  "detail": {"originalSource": ["my.application"]}
}

Note: Status notifications are best-effort. For authoritative status, query the tracking table.
```

### Scenario 8: User receives CloudWatch alarm emails during burst loads

**Symptom:** User receives alarm notification emails like:
- "EmailSender-Errors has entered the ALARM state"

**Important:** With the updated alarm configuration (>50 errors for 2 consecutive 5-minute periods), burst throttling should NOT trigger alarms. If a user receives an alarm, it indicates a real sustained problem.

**What's happening:**
1. Errors exceeded 50 for 2 consecutive 5-minute periods (10+ minutes)
2. This is NOT normal burst behavior - it indicates a real problem
3. The system is experiencing sustained failures

**Response template:**
```
The Lambda error alarm indicates sustained errors over 10+ minutes. This is different from temporary burst throttling - it suggests a real problem.

Let's investigate:
1. Check CloudWatch Logs for error patterns
2. Check if the DLQ has messages
3. Verify SES account status

What error messages are you seeing in CloudWatch Logs?
```

**If user says "but I was just running a load test":**
```
The alarm is configured to ignore temporary burst throttling (it requires >50 errors for 2 consecutive 5-minute periods). If it triggered during your load test, it means:

1. The errors persisted for 10+ minutes (not just an initial burst)
2. OR the retry mechanisms aren't resolving the backlog

Let's check:
- Did all emails eventually deliver?
- Is the EventBridge DLQ empty?
- What's your SES sending rate vs. the volume you submitted?
```

**When to escalate:**
- Alarm stays in ALARM state without recovering
- DLQ has messages
- SES account issues (suspended, reputation problems)

---

## Troubleshooting Response Patterns

These patterns show how to structure your investigation and what information to gather.

### When you find the issue:

```
I found the issue: [ERROR_CODE] - [Brief description]

**What happened:** [Explanation]

**To fix this:**
1. [Step 1]
2. [Step 2]
3. [Step 3]

**To verify the fix:**
[Diagnostic query or test]

For more details, see: [Link to doc section]
```

### When you need more info:

```
To diagnose this, gather:

1. What is your stack name? (e.g., "my-email-engine")
2. Do you have an emailId or recipient email address?
3. What error message are you seeing (if any)?

Once I have this, I can query your tracking data to find the issue.
```

### When the issue is on recipient side:

```
Good news: The email was delivered successfully!

The tracking shows status "delivered" at [timestamp], which means SES confirmed delivery to the recipient's mail server.

If the recipient hasn't received it:
1. Check their spam/junk folder
2. Check if their mail server has additional filtering
3. Ask them to whitelist your sender address

This is outside SESMailEngine's control - the email left AWS successfully.
```

---

## Important Notes

1. **Never suggest deleting suppression entries** without explaining the risk (damages SES reputation)

2. **Bounce rate threshold** default is 5%. AWS suspends accounts at ~10%.

3. **Status priority** means higher statuses can't be overwritten:
   - If status is "opened", it won't change to "delivered"
   - This prevents race conditions

4. **SQS retry behavior**: If bounce rate exceeded or recipient suppressed during SQS retry, a tracking record is created with status="failed" and the message is consumed (no silent loss)

5. **Template cache**: Templates are cached for 5 minutes. Changes may take up to 5 minutes to take effect.

6. **Email IDs**: 
   - Customer-provided IDs are preserved
   - Auto-generated IDs start with "email-"
   - Retry attempts get new IDs but same `originalEmailId`
   - Use `original-email-id-index` GSI to query all attempts for a specific email (see DIAGNOSTIC_QUERIES.md)

7. **Soft bounce handling**:
   - SESMailEngine includes automatic soft bounce retries via AWS SES (up to 12 hours)
   - If delivery ultimately fails, the address is tracked as `soft_bounced`
   - 3 consecutive soft bounces to same address → permanent suppression
   - Customer can resend to `soft_bounced` addresses if appropriate

8. **CloudWatch Alarms**:
   - All alarms notify the AdminEmail address
   - SES alarms are critical - bounce >3% or complaint >0.05% risks account suspension
   - DLQ alarms indicate processing failures that need investigation
   - Lambda error alarms: >50 errors for 2 consecutive 5-minute periods (ignores burst throttling)
   - If Lambda alarm triggers, it's a real sustained problem - investigate immediately

9. **Status Notifications**:
   - Published to EventBridge when status changes (delivered, bounced, complained, opened)
   - Source is `"sesmailengine"`, detail-type is `"Email Status Changed"`
   - `originalSource` field contains the customer's original source value for filtering
   - Notifications are best-effort - query tracking table for authoritative status
   - Consumer services create their own EventBridge rules to subscribe


---

## Performance Reference

When users ask about performance or costs, use these benchmarks:

### Throughput Expectations

| Phase | Rate | Notes |
|-------|------|-------|
| Event submission | 400-500/sec | EventBridge handles bursts well |
| Email processing | SES rate limit | Sandbox: 14/sec, Production: 50-500+/sec |

### Cost Guidance (with Free Tier)

| Volume | Monthly Cost | Notes |
|--------|--------------|-------|
| <62k emails | ~$0.10 | Almost entirely free tier |
| 100k emails | ~$3.90 | Only SES + EventBridge costs |
| 500k emails | ~$45 | SES is main cost driver |

**Key points:**
- Lambda, DynamoDB, S3 typically stay within free tier
- SES: First 62k free from Lambda/EC2, then $0.10/1000
- EventBridge: $1 per million events (no free tier)

### When Users Ask "Is it production ready?"

Answer: Yes. The architecture provides:
- Zero data loss (DynamoDB tracking for every email)
- Automatic retries (EventBridge + SQS)
- Scales with SES limits (request quota increase for higher volume)
- Cost-effective (~85-90% cheaper than SaaS alternatives)
