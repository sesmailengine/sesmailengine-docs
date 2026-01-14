# Setup Guide

This guide walks you through deploying SESEmailEngine to your AWS account.

## Prerequisites

Before deploying, ensure you have:

1. **Python 3.8+** installed on your computer
2. **AWS Account** with admin permissions to create CloudFormation stacks
3. **AWS CLI configured** OR environment variables set (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`)
4. **Verified SES Domain or Email** - At least one email address verified in SES
5. **SES Production Access** (recommended) - Request removal from SES sandbox for production use

### ⚠️ IMPORTANT: SES Region Alignment

**SESMailEngine MUST be deployed in the same AWS region as your SES setup.**

SES is a regional service, which means:
- Verified identities (domains/emails) are region-specific
- Sandbox vs production status is region-specific
- Cross-region SES calls add latency and cost

**Before installing, verify your SES region:**
```bash
# Check which region has your verified identities
aws ses list-identities --region us-east-1
aws ses list-identities --region eu-west-1
# ... check your expected region
```

**Set the correct region before running the installer:**
```bash
# Option 1: Environment variable
export AWS_REGION=eu-west-1

# Option 2: AWS CLI config
aws configure set region eu-west-1
```

### Check Prerequisites

```bash
# Check Python version (need 3.8+)
python --version

# Check AWS credentials are configured
aws sts get-caller-identity
```

---

## Quick Start (Recommended)

The easiest way to deploy SESEmailEngine is using the Python installer.

### Step 1: Download and Extract

Download `sesmailengine-v1.0.0.zip` from [sesmailengine.com](https://sesmailengine.com) and extract it:

```bash
unzip sesmailengine-v1.0.0.zip
cd sesmailengine-v1.0.0
```

### Step 2: Install Dependencies

```bash
pip install -r requirements.txt
```

### Step 3: Run the Installer

```bash
python install.py
```

The installer will guide you through the setup process:

```
╔═══════════════════════════════════════════════════════════════╗
║                                                               ║
║   SESMailEngine Installer v1.0.0                              ║
║   Serverless Email Infrastructure for AWS                     ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝

Checking prerequisites...
✓ All prerequisites met
✓ AWS Region: eu-west-2
✓ AWS Account: 123456789012

⚠ IMPORTANT: SES Region Alignment
────────────────────────────────────────
  SESMailEngine will be deployed to: eu-west-2
  Your SES verified identities (domains/emails) must be in the SAME region.

Is eu-west-2 the correct region for your SES setup? [Y/n]: 

Configuration
────────────────────────────────────────
S3 bucket name for Lambda code [sesmailengine-123456789012-eu-west-2]: my-company-sesmailengine
Default sender email (must be verified in SES): noreply@mycompany.com
Admin email for CloudWatch alerts: admin@mycompany.com
Default sender name [SESMailEngine]: My Company
CloudFormation stack name [sesmailengine]: 

Configuration Summary
────────────────────────────────────────
  S3 Bucket:      my-company-sesmailengine
  Sender Email:   noreply@mycompany.com
  Admin Email:    admin@mycompany.com
  Sender Name:    My Company
  Stack Name:     sesmailengine
  Region:         eu-west-2

Proceed with installation? [Y/n]: 

[1/7] Creating S3 bucket...
✓ Created S3 bucket: my-company-sesmailengine

[2/7] Uploading Lambda packages...
  Uploading email-sender.zip...
  Uploading feedback-processor.zip...
  Uploading template-seeder.zip...
✓ Uploaded all Lambda packages

[3/7] Uploading starter templates...
✓ Uploaded 8 starter templates

[4/7] Deploying CloudFormation stack...
✓ Started CloudFormation stack creation: sesmailengine

[5/7] Waiting for stack creation...
  Waiting for stack creation (this takes 2-3 minutes)...
✓ Stack created successfully

[6/7] Retrieving stack outputs...
✓ Retrieved stack configuration

[7/7] Installing starter templates...
✓ Installed 8 starter templates: welcome, password-reset, ...

╔═══════════════════════════════════════════════════════════════╗
║                                                               ║
║   Installation Complete!                                      ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝

Your SESMailEngine is ready to use!

EventBridge Bus ARN:
  arn:aws:events:eu-west-2:123456789012:event-bus/sesmailengine-EmailBus

Template Bucket:
  sesmailengine-templates-123456789012

Next Steps:
  1. Verify your sender email in SES (if not already done)
  2. Attach the IAM policy to your application (see docs/INTEGRATION.md)
  3. Send your first email via EventBridge

Need help? Visit https://sesmailengine.com/support
```

### Step 4: Send Your First Email

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

print(f"Event sent: {response}")
```

---

## Installer Commands

### Command Reference

| Command | Result |
|---------|--------|
| `python install.py` | Interactive install - prompts for all configuration values |
| `python install.py --help` | Show help message with available options |
| `python install.py --version` | Show installer version |
| `python install.py --uninstall` | Uninstall stack named "sesmailengine" (default) |
| `python install.py --uninstall --stack-name mystack` | Uninstall a specific stack by name |
| `python install.py --uninstall --delete-bucket` | Uninstall + also delete the S3 template bucket |

### Region Selection

The installer uses your AWS region in this order of priority:

1. `AWS_REGION` environment variable (if set)
2. `AWS_DEFAULT_REGION` environment variable (if set)
3. Region from your AWS CLI configuration (`~/.aws/config`)

To deploy to a specific region, set it before running the installer:

```bash
# Option 1: Environment variable (recommended)
export AWS_REGION=eu-west-1
python install.py

# Option 2: AWS CLI config
aws configure set region eu-west-1
python install.py
```

The installer will display the detected region and ask you to confirm before proceeding.

### Uninstall

To remove SESMailEngine:

```bash
python install.py --uninstall
```

To uninstall a stack with a custom name:

```bash
python install.py --uninstall --stack-name mystack
```

This deletes the CloudFormation stack. By default, your data is preserved:
- DynamoDB tables (email history, suppression list) are retained
- S3 bucket (templates) is retained

To also delete the S3 bucket:

```bash
python install.py --uninstall --delete-bucket
```

---

## Configuration Parameters

The installer prompts for these values:

### Required Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| S3 Bucket Name | Bucket for Lambda code and templates | `my-company-sesmailengine` |
| Sender Email | Default "From" email (must be verified in SES) | `noreply@mycompany.com` |
| Admin Email | Email for CloudWatch alarm notifications | `admin@mycompany.com` |

### Optional Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| Sender Name | `SESMailEngine` | Display name shown in recipient's inbox |
| Stack Name | `sesmailengine` | CloudFormation stack name |
| Region | From AWS config | AWS region for deployment |

### Advanced Parameters

These can be customized after deployment via CloudFormation:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `Environment` | `prod` | Environment name (`dev`, `staging`, `prod`) |
| `BounceRateThreshold` | `5` | Max bounce rate % before blocking sends (1-10) |
| `DataRetentionDays` | `90` | Days to retain email tracking data (30-365) |
| `EmailSenderMemory` | `512` | Lambda memory in MB (256, 512, 1024) |
| `EmailSenderTimeout` | `30` | Lambda timeout in seconds (10-60) |
| `EmailSenderConcurrency` | `14` | Max concurrent Lambda executions |
| `FeedbackProcessorMemory` | `256` | Lambda memory in MB (128, 256, 512) |
| `FeedbackProcessorTimeout` | `15` | Lambda timeout in seconds (5-30) |
| `RetainDataOnDelete` | `true` | Keep DynamoDB tables and S3 bucket when stack is deleted |

---

## Data Protection (RetainDataOnDelete)

By default, SESMailEngine protects your data when the CloudFormation stack is deleted. This prevents accidental data loss.

### What Gets Protected

When `RetainDataOnDelete=true` (default):

| Resource | Behavior on Stack Delete |
|----------|-------------------------|
| **EmailTracking Table** | ✅ Retained - Your email history is preserved |
| **Suppression Table** | ✅ Retained - Your bounce/complaint list is preserved |
| **Template Bucket** | ✅ Retained - Your custom templates are preserved |
| Lambda Functions | ❌ Deleted |
| EventBridge Bus | ❌ Deleted |
| CloudWatch Alarms | ❌ Deleted |

### Why This Matters

1. **Email History**: The EmailTracking table contains your complete email audit trail. Losing this means losing compliance records.

2. **Suppression List**: The Suppression table contains email addresses that bounced or complained. Losing this and resending to those addresses can get your SES account suspended.

3. **Custom Templates**: If you've customized templates, they're preserved in the S3 bucket.

### Reinstalling After Uninstall

If you uninstall with `RetainDataOnDelete=true` and want to reinstall:

**Option A: Use a different stack name**
```bash
python install.py
# When prompted for stack name, use a different name like "sesmailengine-v2"
```

**Option B: Delete retained resources first (DATA LOSS)**
```bash
# Delete DynamoDB tables
aws dynamodb delete-table --table-name sesmailengine-EmailTracking
aws dynamodb delete-table --table-name sesmailengine-Suppression

# Delete S3 bucket (must empty first)
aws s3 rb s3://sesmailengine-templates-123456789012 --force

# Now reinstall with same stack name
python install.py
```

### Development/Testing Mode

For development environments where data loss is acceptable, set `RetainDataOnDelete=false`:

**During installation**: When prompted "Retain data on stack delete?", answer `n`

**After deployment**: Update the stack parameter:
```bash
aws cloudformation update-stack \
  --stack-name sesmailengine \
  --use-previous-template \
  --parameters \
    ParameterKey=RetainDataOnDelete,ParameterValue=false \
  --capabilities CAPABILITY_NAMED_IAM
```

---

## Branding Configuration

Set `DefaultSenderName` to your company/product name for proper branding:

| Configuration | Recipient Sees |
|---------------|----------------|
| `SESMailEngine` (default) | `"SESMailEngine" <noreply@yourcompany.com>` |
| `Acme Corp` | `"Acme Corp" <noreply@yourcompany.com>` |
| `My SaaS Product` | `"My SaaS Product" <noreply@yourcompany.com>` |

---

## SES Sending Rate Configuration

The `EmailSenderConcurrency` parameter controls how many emails can be sent simultaneously. This should match your SES account's `MaxSendRate` to prevent throttling errors.

**Check your SES quota:**
```bash
aws ses get-send-quota --region YOUR_REGION
```

Example output:
```json
{
    "Max24HourSend": 50000.0,
    "MaxSendRate": 14.0,
    "SentLast24Hours": 0.0
}
```

**Set concurrency to match your MaxSendRate:**

| SES Account Status | Typical MaxSendRate | Recommended Concurrency |
|--------------------|---------------------|------------------------|
| Sandbox (new accounts) | 1/sec | `1` |
| Production (just approved) | 14/sec | `14` (default) |
| Scaled production | 50-500/sec | Match your MaxSendRate |

**To update after deployment:**
```bash
aws cloudformation update-stack \
  --stack-name sesmailengine \
  --use-previous-template \
  --parameters \
    ParameterKey=EmailSenderConcurrency,ParameterValue=50 \
  --capabilities CAPABILITY_NAMED_IAM
```

---

## Post-Installation Steps

### Verify SES Configuration

After deployment, verify your SES setup:

1. **Check Configuration Set** was created:
   ```bash
   aws ses describe-configuration-set \
     --configuration-set-name sesmailengine-ConfigSet
   ```

2. **Verify sender email** is verified in SES:
   ```bash
   aws ses get-identity-verification-attributes \
     --identities noreply@yourcompany.com
   ```

### Grant Your Services Access

Your applications need IAM permissions to publish events. Add this policy to your service's IAM role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "events:PutEvents",
      "Resource": "arn:aws:events:REGION:ACCOUNT_ID:event-bus/sesmailengine-EmailBus"
    }
  ]
}
```

Get the exact ARN from stack outputs:
```bash
aws cloudformation describe-stacks \
  --stack-name sesmailengine \
  --query 'Stacks[0].Outputs[?OutputKey==`EventBusArn`].OutputValue' \
  --output text
```

See [INTEGRATION.md](INTEGRATION.md) for detailed IAM setup instructions.

### Subscribe to Email Status Notifications (Optional)

SESEmailEngine publishes "Email Status Changed" events when emails are delivered, bounced, or marked as spam. Your services can subscribe to these events to:
- Update user records when emails bounce
- Track delivery confirmations
- Handle spam complaints immediately

Create an EventBridge rule to receive notifications:

```bash
aws events put-rule \
  --name "my-app-email-notifications" \
  --event-bus-name "sesmailengine-EmailBus" \
  --event-pattern '{
    "source": ["sesmailengine"],
    "detail-type": ["Email Status Changed"],
    "detail": {
      "originalSource": ["my.application"]
    }
  }'
```

The `originalSource` field matches the `source` you used when sending emails, so you only receive notifications for your own emails.

See [INTEGRATION.md](INTEGRATION.md#receiving-status-notifications) for complete examples including Lambda handlers and IAM policies.

---

## Stack Outputs

After deployment, these outputs are available:

| Output | Description |
|--------|-------------|
| `EventBusName` | EventBridge bus name for sending emails |
| `EventBusArn` | EventBridge bus ARN for IAM policies |
| `TemplateBucketName` | S3 bucket for email templates |
| `EmailTrackingTableName` | DynamoDB table for email logs |
| `SuppressionTableName` | DynamoDB table for suppression list |
| `CustomerIAMPolicy` | Ready-to-use IAM policy JSON |
| `EmailSenderLambdaArn` | Email Sender Lambda function ARN |
| `FeedbackProcessorLambdaArn` | Feedback Processor Lambda function ARN |
| `TemplateSeederLambdaName` | Template Seeder Lambda function name |
| `InstallTemplatesCommand` | AWS CLI command to install starter templates |
| `AlarmNotificationTopicArn` | SNS topic for CloudWatch alarm notifications |
| `RetryQueueUrl` | SQS queue URL for email retries |
| `SESConfigurationSetName` | SES configuration set name |

View all outputs:
```bash
aws cloudformation describe-stacks \
  --stack-name sesmailengine \
  --query 'Stacks[0].Outputs'
```

---

## Updating the Stack

To update parameters after deployment:

```bash
aws cloudformation update-stack \
  --stack-name sesmailengine \
  --use-previous-template \
  --parameters \
    ParameterKey=DefaultSenderName,ParameterValue="New Company Name" \
  --capabilities CAPABILITY_NAMED_IAM
```

---

## Starter Templates

The installer automatically installs 8 production-ready email templates:

| Template | Description |
|----------|-------------|
| `welcome` | New user registration welcome email |
| `password-reset` | Secure password reset with expiration |
| `purchase-confirmation` | Order/purchase confirmation receipt |
| `email-verification` | Email address verification |
| `invoice` | Professional invoice with line items |
| `subscription-renewal` | Subscription renewal reminder |
| `contact-form-notification` | Internal contact form notification |
| `shipping-notification` | Order shipped with tracking |

All templates feature responsive design, optional logo support, and plain text fallbacks.

See [TEMPLATES.md](TEMPLATES.md) for template customization.

---

## CloudWatch Alarms

The stack creates several CloudWatch alarms to monitor system health:

| Alarm | Trigger | Action |
|-------|---------|--------|
| EventBridge DLQ Depth | Messages in DLQ > 0 | Check failed email requests |
| Retry DLQ Depth | Messages in DLQ > 0 | Check failed retry attempts |
| Feedback DLQ Depth | Messages in DLQ > 0 | Check failed feedback processing |
| Email Sender Errors | Lambda errors > 50 (sustained) | Check Lambda logs |
| Feedback Processor Errors | Lambda errors > 50 (sustained) | Check Lambda logs |
| SES Bounce Rate | Bounce rate > 3% | Clean email list, review sending practices |
| SES Complaint Rate | Complaint rate > 0.05% | Review email content and targeting |

All alarms send notifications to the email address specified during installation.

**Important:** SES bounce rate alarms trigger at 3% (AWS suspends accounts at ~5%). Complaint rate alarms trigger at 0.05% (AWS suspends at ~0.1%). Monitor these closely to protect your SES reputation.

---

## Troubleshooting Installation

### Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| "Bucket name already exists" | S3 bucket names are globally unique | Try a different name with your company prefix |
| "Email address is not verified" | Sender email not verified in SES | Verify the email in SES console first |
| "Access Denied" | AWS credentials lack permissions | Use credentials with admin access |
| "Stack already exists" | Stack name already in use | Use a different stack name or delete existing stack |

### Installer Logs

The installer creates a log file for debugging:

```bash
cat sesmailengine-install.log
```

### Manual Cleanup After Failed Install

If installation fails partway through:

```bash
# Delete partial CloudFormation stack
aws cloudformation delete-stack --stack-name sesmailengine

# Delete S3 bucket (if created)
aws s3 rb s3://my-company-sesmailengine --force
```

---

## Advanced: Manual Deployment (CLI)

For users who prefer manual deployment or need custom configurations.

**Important:** Replace `YOUR_REGION` with the AWS region where your SES identities are verified (e.g., `us-east-1`, `eu-west-1`).

**Package Structure:** After extracting `sesmailengine-v1.0.0.zip`, you'll have:
```
sesmailengine-v1.0.0/
├── install.py           # Automated installer
├── requirements.txt     # Python dependencies (boto3)
├── template.yaml        # CloudFormation template
├── README.md
├── docs/                # Documentation
├── lambda/              # Lambda ZIP packages
│   ├── email-sender.zip
│   ├── feedback-processor.zip
│   └── template-seeder.zip
└── starter-templates/   # Email templates
    ├── welcome/
    ├── password-reset/
    └── ... (8 template folders)
```

Run all commands from inside the `sesmailengine-v1.0.0` folder.

### Step 1: Create S3 Bucket

```bash
aws s3 mb s3://my-company-sesmailengine --region YOUR_REGION
```

### Step 2: Upload Files

```bash
# Upload Lambda packages
aws s3 cp lambda/email-sender.zip s3://my-company-sesmailengine/lambda/
aws s3 cp lambda/feedback-processor.zip s3://my-company-sesmailengine/lambda/
aws s3 cp lambda/template-seeder.zip s3://my-company-sesmailengine/lambda/

# Upload starter templates
aws s3 cp starter-templates/ s3://my-company-sesmailengine/starter-templates/ --recursive
```

### Step 3: Deploy CloudFormation

```bash
aws cloudformation create-stack \
  --stack-name sesmailengine \
  --template-body file://template.yaml \
  --region YOUR_REGION \
  --parameters \
    ParameterKey=LambdaCodeS3Bucket,ParameterValue=my-company-sesmailengine \
    ParameterKey=AdminEmail,ParameterValue=admin@mycompany.com \
    ParameterKey=DefaultSenderEmail,ParameterValue=noreply@mycompany.com \
    ParameterKey=DefaultSenderName,ParameterValue="My Company" \
  --capabilities CAPABILITY_NAMED_IAM

# Wait for completion
aws cloudformation wait stack-create-complete --stack-name sesmailengine --region YOUR_REGION
```

### Step 4: Install Templates

```bash
aws lambda invoke \
  --function-name sesmailengine-TemplateSeeder \
  --region YOUR_REGION \
  --output text /dev/stdout
```

---

## Advanced: Manual Deployment (AWS Console)

For users who prefer using the AWS web console instead of CLI.

**Important:** Deploy to the same region where your SES identities (domains/emails) are verified.

### Step 1: Create S3 Bucket

1. Log in to the [AWS Console](https://console.aws.amazon.com/)
2. Make sure you're in the correct region (check top-right corner, e.g., "EU (Ireland)")
3. Go to **S3** (search "S3" in the search bar)
4. Click **Create bucket**
5. Enter a bucket name: `my-company-sesmailengine` (must be globally unique)
6. Leave all other settings as default
7. Click **Create bucket**

### Step 2: Upload Lambda Packages

1. Click on your newly created bucket to open it
2. Click **Create folder**, name it `lambda`, click **Create folder**
3. Click into the `lambda` folder
4. Click **Upload**
5. Click **Add files** and select these 3 files from your extracted `sesmailengine-v1.0.0/lambda/` folder:
   - `email-sender.zip`
   - `feedback-processor.zip`
   - `template-seeder.zip`
6. Click **Upload**
7. Wait for upload to complete, then click **Close**

### Step 3: Upload Starter Templates

1. Go back to your bucket root (click bucket name in breadcrumb)
2. Click **Create folder**, name it `starter-templates`, click **Create folder**
3. Click into the `starter-templates` folder
4. For each template folder (`welcome`, `password-reset`, `purchase-confirmation`, `email-verification`, `invoice`, `subscription-renewal`, `contact-form-notification`, `shipping-notification`):
   - Click **Create folder** with the template name (e.g., `welcome`)
   - Click into that folder
   - Click **Upload** → **Add files**
   - Select all 3 files from that template folder (`template.html`, `template.txt`, `metadata.json`)
   - Click **Upload**, then **Close**
   - Go back to `starter-templates` folder and repeat for each template

**Tip:** This step is tedious via console. Consider using AWS CLI just for this step:
```bash
aws s3 cp starter-templates/ s3://my-company-sesmailengine/starter-templates/ --recursive
```

### Step 4: Deploy CloudFormation Stack

1. Go to **CloudFormation** (search "CloudFormation" in the search bar)
2. Click **Create stack** → **With new resources (standard)**
3. Select **Upload a template file**
4. Click **Choose file** and select `template.yaml` from your extracted `sesmailengine-v1.0.0` folder
5. Click **Next**
6. Enter stack details:
   - **Stack name:** `sesmailengine`
   - **LambdaCodeS3Bucket:** `my-company-sesmailengine` (your bucket name from Step 1)
   - **AdminEmail:** `admin@mycompany.com` (your email for alerts)
   - **DefaultSenderEmail:** `noreply@mycompany.com` (must be verified in SES)
   - **DefaultSenderName:** `My Company` (or your company name)
   - Leave other parameters as defaults
7. Click **Next**
8. On Configure stack options, leave defaults and click **Next**
9. On Review page:
   - Scroll to bottom
   - Check the box: **I acknowledge that AWS CloudFormation might create IAM resources with custom names**
   - Click **Submit**
10. Wait for stack status to change to **CREATE_COMPLETE** (2-3 minutes)
    - Click the refresh button to update status

### Step 5: Install Starter Templates

1. Go to **Lambda** (search "Lambda" in the search bar)
2. Find and click on `sesmailengine-TemplateSeeder`
3. Click the **Test** tab
4. Click **Test** button (you can use the default test event)
5. You should see a success response showing installed templates

### Step 6: Verify Deployment

1. Go to **CloudFormation** → click your `sesmailengine` stack
2. Click the **Outputs** tab
3. Note down these values for integration:
   - `EventBusName` - for sending emails
   - `EventBusArn` - for IAM policies
   - `TemplateBucketName` - for uploading custom templates

Your SESMailEngine is now deployed and ready to use!

---

## Next Steps

- [TEMPLATES.md](TEMPLATES.md) - Create and customize email templates
- [INTEGRATION.md](INTEGRATION.md) - Integration examples for different languages
- [BOUNCE_RATE_PROTECTION.md](BOUNCE_RATE_PROTECTION.md) - Understand bounce rate monitoring
- [SECURITY.md](SECURITY.md) - Security architecture and compliance
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) - Common issues and solutions
