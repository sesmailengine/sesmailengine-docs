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
╔══════════════════════════════════════════════════════════════╗
║           SESMailEngine Installer v1.0.0                     ║
╚══════════════════════════════════════════════════════════════╝

Step 1/7: Checking prerequisites...
  ✓ Python 3.11 detected
  ✓ AWS credentials configured
  ✓ Region: eu-west-2

Step 2/7: Collecting configuration...
  Enter S3 bucket name for deployment: my-company-sesmailengine
  Enter your verified SES email: noreply@mycompany.com
  Enter admin email for alerts: admin@mycompany.com
  Enter sender display name [SESMailEngine]: My Company
  Enter stack name [sesmailengine]: 

Step 3/7: Creating S3 bucket...
  ✓ Bucket 'my-company-sesmailengine' created in eu-west-2

Step 4/7: Uploading files...
  ✓ Uploaded lambda/email-sender.zip
  ✓ Uploaded lambda/feedback-processor.zip
  ✓ Uploaded lambda/template-seeder.zip
  ✓ Uploaded 8 starter templates

Step 5/7: Deploying CloudFormation stack...
  Creating stack 'sesmailengine'...
  ⏳ Waiting for stack creation (this takes 2-3 minutes)...
  ✓ Stack created successfully!

Step 6/7: Installing starter templates...
  ✓ Installed 8 email templates

Step 7/7: Complete!

╔══════════════════════════════════════════════════════════════╗
║                    Installation Complete!                     ║
╚══════════════════════════════════════════════════════════════╝

Your SESMailEngine is ready to use!

EventBridge Bus: sesmailengine-EmailBus
Template Bucket: sesmailengine-templates-123456789012

Next steps:
1. Grant your applications access (see docs/INTEGRATION.md)
2. Send your first email (see below)

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

## Installer Options

### View Help

```bash
python install.py --help
```

### Non-Interactive Mode

For CI/CD or scripted deployments:

```bash
python install.py \
  --bucket my-company-sesmailengine \
  --sender-email noreply@mycompany.com \
  --admin-email admin@mycompany.com \
  --sender-name "My Company" \
  --stack-name sesmailengine \
  --region eu-west-2
```

### Uninstall

To remove SESMailEngine:

```bash
python install.py --uninstall --stack-name sesmailengine
```

This deletes the CloudFormation stack. By default, your data is preserved:
- DynamoDB tables (email history, suppression list) are retained
- S3 bucket (templates) is retained

To also delete the S3 bucket:

```bash
python install.py --uninstall --stack-name sesmailengine --delete-bucket
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

## Advanced: Manual Deployment

For users who prefer manual deployment or need custom configurations.

### Step 1: Create S3 Bucket

```bash
aws s3 mb s3://my-company-sesmailengine --region eu-west-2
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
  --parameters \
    ParameterKey=LambdaCodeS3Bucket,ParameterValue=my-company-sesmailengine \
    ParameterKey=AdminEmail,ParameterValue=admin@mycompany.com \
    ParameterKey=DefaultSenderEmail,ParameterValue=noreply@mycompany.com \
    ParameterKey=DefaultSenderName,ParameterValue="My Company" \
  --capabilities CAPABILITY_NAMED_IAM

# Wait for completion
aws cloudformation wait stack-create-complete --stack-name sesmailengine
```

### Step 4: Install Templates

```bash
aws lambda invoke \
  --function-name sesmailengine-TemplateSeeder \
  --output text /dev/stdout
```

---

## Next Steps

- [TEMPLATES.md](TEMPLATES.md) - Create and customize email templates
- [INTEGRATION.md](INTEGRATION.md) - Integration examples for different languages
- [BOUNCE_RATE_PROTECTION.md](BOUNCE_RATE_PROTECTION.md) - Understand bounce rate monitoring
- [SECURITY.md](SECURITY.md) - Security architecture and compliance
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) - Common issues and solutions
