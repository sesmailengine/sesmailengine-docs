# Security & Compliance

This document details the security architecture, data protection measures, and compliance considerations for SESMailEngine.

---

## Architecture Security Overview

SESMailEngine runs entirely within your AWS account using a serverless architecture. No data leaves your account, and there are no external API calls or third-party dependencies.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         YOUR AWS ACCOUNT                                    │
│                                                                             │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│   │ EventBridge │───▶│   Lambda    │───▶│     SES     │───▶│  Recipient  │  │
│   │  (TLS 1.2+) │    │ (Encrypted) │    │ (TLS Req.)  │    │             │  │
│   └─────────────┘    └──────┬──────┘    └─────────────┘    └─────────────┘  │
│                             │                                               │
│                    ┌────────┴────────┐                                      │
│                    ▼                 ▼                                      │
│             ┌─────────────┐   ┌─────────────┐                               │
│             │  DynamoDB   │   │     S3      │                               │
│             │ (SSE-KMS)   │   │ (SSE-S3)    │                               │
│             └─────────────┘   └─────────────┘                               │
│                                                                             │
│   ✓ All data encrypted at rest                                              │
│   ✓ All traffic encrypted in transit (TLS 1.2+)                             │
│   ✓ No external API calls                                                   │
│   ✓ Least privilege IAM policies                                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Data Encryption

### Encryption at Rest

All data stored by SESMailEngine is encrypted at rest using AWS-managed encryption:

| Service | Encryption Method | Key Management |
|---------|-------------------|----------------|
| DynamoDB (EmailTracking) | SSE with AWS-managed keys | AWS KMS (automatic) |
| DynamoDB (Suppression) | SSE with AWS-managed keys | AWS KMS (automatic) |
| S3 (Templates) | AES-256 with S3 Bucket Keys | AWS-managed |
| SQS (Email Queue) | SSE with SQS-managed keys | AWS-managed |
| SQS (Dead Letter Queues) | SSE with SQS-managed keys | AWS-managed |

**CloudFormation Configuration:**
```yaml
# DynamoDB Tables
SSESpecification:
  SSEEnabled: true

# S3 Bucket
BucketEncryption:
  ServerSideEncryptionConfiguration:
    - ServerSideEncryptionByDefault:
        SSEAlgorithm: AES256
      BucketKeyEnabled: true

# SQS Queues
SqsManagedSseEnabled: true
```

### Encryption in Transit

All data in transit is encrypted using TLS 1.2 or higher:

| Communication Path | Encryption |
|-------------------|------------|
| Customer App → EventBridge | TLS 1.2+ (AWS SDK) |
| Lambda → DynamoDB | TLS 1.2+ (AWS SDK) |
| Lambda → S3 | TLS 1.2+ (AWS SDK) |
| Lambda → SES | TLS 1.2+ (AWS SDK) |
| SES → Recipient Mail Server | TLS Required (enforced) |
| S3 Bucket Access | HTTPS only (bucket policy enforced) |

**SES TLS Enforcement:**
```yaml
SESConfigurationSet:
  DeliveryOptions:
    TlsPolicy: REQUIRE  # Emails only sent over TLS connections
```

**S3 HTTPS-Only Policy:**
```yaml
TemplateBucketPolicy:
  PolicyDocument:
    Statement:
      - Sid: DenyInsecureTransport
        Effect: Deny
        Principal: '*'
        Action: 's3:*'
        Condition:
          Bool:
            aws:SecureTransport: 'false'
```

---

## IAM & Access Control

### Principle of Least Privilege

All Lambda functions operate with minimal required permissions. No wildcard (`*`) permissions are used.

**Email Sender Lambda Permissions:**
- CloudWatch Logs: Write to specific log group only
- DynamoDB EmailTracking: PutItem, UpdateItem, GetItem, Query
- DynamoDB Suppression: GetItem, Query (read-only)
- S3 Templates: GetObject, ListBucket (prefix-restricted)
- SES: SendEmail, SendRawEmail (identity-scoped)
- SQS: ReceiveMessage, DeleteMessage (specific queue)

**Feedback Processor Lambda Permissions:**
- CloudWatch Logs: Write to specific log group only
- DynamoDB EmailTracking: UpdateItem, Query
- DynamoDB Suppression: PutItem, UpdateItem, GetItem, Query
- SQS: SendMessage (specific queue)

**Template Seeder Lambda Permissions:**
- CloudWatch Logs: Write to specific log group only
- S3 Source: GetObject, ListBucket (source bucket)
- S3 Destination: PutObject, GetObject (template bucket)

### No Cross-Account Access

- All IAM roles use `sts:AssumeRole` restricted to `lambda.amazonaws.com`
- No cross-account trust relationships
- No external principal access

### Customer Integration Security

Customers grant their services access via a minimal IAM policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "events:PutEvents",
      "Resource": "arn:aws:events:REGION:ACCOUNT:event-bus/STACK-EmailBus"
    }
  ]
}
```

This policy grants only the ability to publish events—no read access to email data, templates, or logs.

---

## Data Privacy & Retention

### Data Isolation

- **No external data transmission**: All processing occurs within your AWS account
- **No third-party services**: SESMailEngine has no external dependencies
- **No telemetry or analytics**: We do not collect any usage data
- **Customer-controlled data**: You own and control all email data

### Data Retention (GDPR Compliance)

Email tracking data is automatically deleted after a configurable retention period:

| Parameter | Default | Range | Purpose |
|-----------|---------|-------|---------|
| `DataRetentionDays` | 90 days | 30-365 days | GDPR Article 5(1)(e) compliance |
| `LogRetentionDays` | 90 days | 1-3653 days | CloudWatch log retention (cost optimization) |

**Implementation:**
- DynamoDB TTL automatically deletes records after the retention period
- CloudWatch logs are automatically deleted after the log retention period
- No manual intervention required
- Configurable at deployment time

```yaml
TimeToLiveSpecification:
  AttributeName: ttl
  Enabled: true
```

### Data Stored

| Table | Data Stored | Retention |
|-------|-------------|-----------|
| EmailTracking | Email ID, recipient, subject, status, timestamps | Configurable (default 90 days) |
| Suppression | Email address, suppression reason, date | Permanent (required for compliance) |
| CloudWatch Logs | Lambda execution logs, errors, debug info | Configurable (default 90 days) |

**Note:** Email content (body) is NOT stored. Only metadata is retained for tracking purposes.

### Point-in-Time Recovery

Both DynamoDB tables have Point-in-Time Recovery (PITR) enabled:

```yaml
PointInTimeRecoverySpecification:
  PointInTimeRecoveryEnabled: true
```

This allows recovery of data to any point within the last 35 days in case of accidental deletion.

---

## Regulatory Compliance

### Shared Responsibility Model

SESMailEngine operates under the AWS Shared Responsibility Model:

| Responsibility | Owner |
|----------------|-------|
| Physical security of data centers | AWS |
| Network infrastructure security | AWS |
| Hypervisor and host security | AWS |
| Service-level encryption | AWS (via SESMailEngine configuration) |
| IAM policies and access control | Customer (via SESMailEngine defaults) |
| Email content and recipient consent | Customer |
| Data subject requests (GDPR) | Customer |

### GDPR (EU) & UK GDPR Compliance

SESMailEngine supports GDPR compliance through:

| GDPR Requirement | SESMailEngine Feature |
|------------------|----------------------|
| **Article 5(1)(e)** - Storage limitation | Configurable data retention (30-365 days) |
| **Article 5(1)(f)** - Integrity and confidentiality | Encryption at rest and in transit |
| **Article 17** - Right to erasure | Suppression list + DynamoDB TTL |
| **Article 25** - Data protection by design | Minimal data collection|
| **Article 32** - Security of processing | AWS encryption, IAM least privilege |
| **Article 33** - Breach notification | CloudWatch alarms, CloudTrail logging |

**Customer Responsibilities:**
- Obtain valid consent before sending marketing emails
- Honor unsubscribe requests (add to suppression list)
- Respond to data subject access requests
- Maintain records of processing activities

### US Compliance Considerations

| Regulation | Applicability | Notes |
|------------|---------------|-------|
| **CAN-SPAM** | Marketing emails to US recipients | Customer must include unsubscribe mechanism |
| **CCPA** | California residents | Customer must honor opt-out requests |
| **HIPAA** | Healthcare data | Not recommended without additional controls |

**Note:** SESMailEngine does not store email content, which simplifies compliance for most use cases. However, customers sending sensitive data should implement additional controls.

### AWS Compliance Certifications

SESMailEngine inherits AWS compliance certifications for the underlying services:

- SOC 1, SOC 2, SOC 3
- ISO 27001, ISO 27017, ISO 27018
- PCI DSS Level 1
- FedRAMP (varies by region)
- GDPR

See [AWS Compliance Programs](https://aws.amazon.com/compliance/programs/) for the complete list.

---

## Network Security

### No VPC Required

SESMailEngine uses AWS managed services that operate outside VPCs:
- Lambda (can optionally be VPC-attached)
- DynamoDB (VPC endpoints available)
- S3 (VPC endpoints available)
- SES (public endpoint)
- EventBridge (public endpoint)

All communication uses AWS's internal network with TLS encryption.

### Optional VPC Configuration

For enhanced network isolation, you can:
1. Deploy Lambda functions in a VPC
2. Use VPC endpoints for DynamoDB and S3
3. Use AWS PrivateLink for SES (where available)

**Note:** VPC configuration is not included in the default deployment and requires manual setup.

### Public Access Blocked

The S3 template bucket blocks all public access:

```yaml
PublicAccessBlockConfiguration:
  BlockPublicAcls: true
  BlockPublicPolicy: true
  IgnorePublicAcls: true
  RestrictPublicBuckets: true
```

---

## Audit & Monitoring

### CloudWatch Logging

All Lambda functions log to CloudWatch Logs:
- Structured JSON logging
- Log retention follows AWS defaults (configurable)
- No sensitive data (email content, PII) in logs

### CloudTrail Integration

AWS CloudTrail automatically captures:
- DynamoDB API calls
- S3 API calls
- Lambda invocations
- SES API calls
- IAM changes

Enable CloudTrail in your account for complete audit trails.

### CloudWatch Alarms

SESMailEngine includes pre-configured alarms for security-relevant events:

| Alarm | Trigger | Security Relevance |
|-------|---------|-------------------|
| DLQ Depth | Messages in dead letter queue | Potential attack or misconfiguration |
| Lambda Errors | Sustained error rate | Potential service disruption |
| SES Bounce Rate | >3% bounce rate | Reputation attack or list quality issue |
| SES Complaint Rate | >0.05% complaint rate | Potential spam complaints |

### Suppression List Audit

Query the suppression list for compliance audits:

```bash
aws dynamodb scan \
  --table-name STACK-Suppression \
  --projection-expression "email,suppressionType,reason,addedToSuppressionDate"
```

---

## Security Best Practices

### Recommendations

1. **Enable CloudTrail** in your AWS account for complete audit logging
2. **Use AWS Organizations SCPs** to restrict regions and services
3. **Enable AWS Config** to monitor resource compliance
4. **Review IAM policies** periodically using IAM Access Analyzer
5. **Monitor CloudWatch alarms** and respond to alerts promptly
6. **Verify SES identities** using DKIM and SPF for email authentication
7. **Use dedicated SES identities** for SESMailEngine (not shared with other applications)

### Email Authentication (Customer Responsibility)

Configure these DNS records for your sending domain:

| Record Type | Purpose |
|-------------|---------|
| **SPF** | Authorizes SES to send on your behalf |
| **DKIM** | Cryptographically signs emails |
| **DMARC** | Policy for handling authentication failures |

See [AWS SES Email Authentication](https://docs.aws.amazon.com/ses/latest/dg/email-authentication.html) for setup instructions.

### Incident Response

If you suspect a security incident:

1. **Check CloudWatch Alarms** for anomalies
2. **Review CloudTrail logs** for unauthorized API calls
3. **Check DLQ messages** for failed processing
4. **Review SES sending statistics** for unusual patterns
5. **Contact AWS Support** if you suspect account compromise

---

## Security Updates

SESMailEngine uses AWS managed services that are automatically patched by AWS. Lambda runtime updates are applied automatically when you redeploy.

To update Lambda runtimes:
1. Download the latest SESMailEngine release
2. Run `python install.py` to redeploy

---

## Contact

For security concerns or vulnerability reports, contact: security@sesmailengine.com

---

*Document version: 1.0.0 | Last updated: January 2026*
