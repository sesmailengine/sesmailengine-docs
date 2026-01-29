# Bounce Rate Protection

SESMailEngine includes automatic bounce and complaint rate monitoring to protect your SES reputation.

## Overview

Amazon SES monitors your bounce and complaint rates and can suspend your sending privileges if they exceed acceptable thresholds. SESMailEngine monitors your SES reputation metrics via CloudWatch alarms and alerts you before AWS takes action.

**Important:** Reputation monitoring is **alarm-only** - emails are NOT blocked based on bounce or complaint rates. This gives you visibility into potential issues while maintaining email delivery.

## How It Works

1. **CloudWatch monitors** your SES account's `Reputation.BounceRate` and `Reputation.ComplaintRate` metrics
2. **If thresholds are exceeded** for 2 consecutive hours, an alarm triggers
3. **Alarm notification** is sent to the AdminEmail SNS topic
4. **Emails continue to send** - no blocking occurs

## AWS SES Thresholds

### Bounce Rate

| Bounce Rate | AWS Action |
|-------------|------------|
| < 5% | Normal operation |
| 5% | Account placed under review |
| 10% | Sending may be paused |

SESMailEngine's alarm triggers at **3%** by default as an early warning.

### Complaint Rate

| Complaint Rate | AWS Action |
|----------------|------------|
| < 0.1% | Normal operation |
| 0.1% | Account placed under review |
| 0.5% | Sending may be paused |

SESMailEngine's alarm triggers at **0.05%** by default as an early warning.

## Configuration

### CloudFormation Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `BounceRateAlarmThreshold` | 0.03 (3%) | Bounce rate threshold for alarm |
| `ComplaintRateAlarmThreshold` | 0.0005 (0.05%) | Complaint rate threshold for alarm |
| `ReputationAlarmPeriod` | 3600 (1 hour) | Evaluation period in seconds |

### Alarm Evaluation

Both alarms use 2 consecutive evaluation periods to reduce noise from temporary spikes.

## Monitoring

### CloudWatch Alarms

| Alarm | Metric | Default Threshold |
|-------|--------|-------------------|
| `{StackName}-SES-BounceRate` | SES Reputation.BounceRate | > 3% for 2 consecutive hours |
| `{StackName}-SES-ComplaintRate` | SES Reputation.ComplaintRate | > 0.05% for 2 consecutive hours |

Alarm notifications are sent to the `{StackName}-Alarms` SNS topic, which emails the `AdminEmail` address.

### Checking Current Rates

View your current SES reputation metrics in the AWS Console:
1. Go to **Amazon SES** → **Account dashboard**
2. Check **Reputation metrics** section

Or via CLI:
```bash
aws ses get-send-statistics
```

## Best Practices

1. **Act on alarms promptly**: When you receive an alarm, investigate immediately
2. **Monitor trends**: Watch for gradual increases before hitting threshold
3. **Clean your lists**: Regularly remove invalid addresses
4. **Use double opt-in**: Reduces invalid addresses from the start
5. **Check SES console**: Compare with SES account-level metrics

## Responding to Alarms

When you receive a bounce rate alarm:

1. **Check recent bounces** in the Suppression table
2. **Identify the source** - which campaign or application is causing bounces?
3. **Review email lists** - are you sending to old or unverified addresses?
4. **Check SES console** - view detailed bounce metrics
5. **Take action** - pause problematic campaigns, clean lists, or investigate further

## Troubleshooting

### High Bounce Rate

1. Check recent bounces in Suppression table:
   ```bash
   aws dynamodb query \
     --table-name ${STACK_NAME}-Suppression \
     --index-name suppression-type-date-index \
     --key-condition-expression "suppressionType = :type" \
     --expression-attribute-values '{":type":{"S":"bounce"}}'
   ```

2. Review bounce reasons - are they mostly `NoEmail` (bad addresses) or `MailboxFull` (temporary)?

3. Check SES console for account-level bounce rate

4. Consider pausing campaigns until you clean your email lists

## Architecture

```
┌─────────────────┐     ┌──────────────────┐
│  AWS SES        │────▶│  CloudWatch      │
│  Reputation     │     │  Metrics         │
│  Metrics        │     └────────┬─────────┘
└─────────────────┘              │
                    ┌────────────┴────────────┐
                    │                         │
              Rate < 3%                  Rate >= 3%
              (2 hours)                  (2 hours)
                    │                         │
                    ▼                         ▼
            ┌───────────────┐       ┌─────────────────┐
            │ No Action     │       │ Trigger Alarm   │
            │               │       │ → SNS → Email   │
            └───────────────┘       └─────────────────┘
                                             │
                                             ▼
                                    ┌─────────────────┐
                                    │ Admin Reviews   │
                                    │ Takes Action    │
                                    └─────────────────┘
```

**Note:** Emails continue to send regardless of bounce rate. The alarm is informational only.

## Soft Bounce Handling

SESMailEngine handles soft bounces (temporary delivery failures) with a consecutive bounce approach.

### Consecutive Soft Bounce Suppression

When a soft bounce occurs, the system checks if the last N emails to that address ALL soft bounced:

| Consecutive Soft Bounces | Action |
|--------------------------|--------|
| 1-2 | Update status to "soft_bounced", no suppression |
| 3+ (configurable) | Add to suppression list |

**Note:**  SESMailEngine retries soft bounces for up to 12 hours via SES before soft bounce event is published.  

### Configuration

The consecutive soft bounce threshold is configurable via CloudFormation parameter:

| Parameter | Default | Range | Description |
|-----------|---------|-------|-------------|
| `ConsecutiveSoftBounceThreshold` | 3 | 2-10 | Consecutive soft bounces before suppression |

### Understanding "soft_bounced" Status

| Status | Meaning | Suppressed? |
|--------|---------|-------------|
| `soft_bounced` | Temporary failure recorded | Only if consecutive threshold reached |
| `bounced` | Hard bounce, address permanently suppressed | Yes |

### Soft Bounce Types

| Bounce SubType | Description | Suppression |
|----------------|-------------|-------------|
| `MailboxFull` | Recipient's mailbox is full | After 3 consecutive |
| `MessageTooLarge` | Email too large for recipient | After 3 consecutive |
| `ContentRejected` | Spam filter rejection | After 3 consecutive |
| `General` | Unspecified temporary failure | After 3 consecutive |
| `NoEmail` | Address doesn't exist | Immediate (hard bounce) |
| `Suppressed` | On SES suppression list | Immediate (hard bounce) |
| `EmailValidationSuppressed` | AWS Auto Validation blocked | Immediate (hard bounce) |


## AWS SES Email Validation Compatibility

SESMailEngine is fully compatible with AWS SES Email Validation (Auto Validation), a feature launched in January 2026 that proactively validates email addresses before sending.

### How It Works Together

When AWS Auto Validation is enabled at the account level:

1. **You send email** through SESMailEngine
2. **SES accepts the request** and returns a MessageId
3. **SES Auto Validation checks** the recipient address
4. **If invalid**, SES generates a bounce notification with `bounceSubType: "EmailValidationSuppressed"`
5. **SESMailEngine receives** the bounce via SNS
6. **Feedback Processor** handles it as a hard bounce:
   - Updates tracking status to "bounced"
   - **Adds address to suppression list immediately**
   - Publishes "Email Status Changed" event
7. **Future sends to this address are blocked** by SESMailEngine's suppression check (no additional validation charges)

### Supported Bounce SubTypes

| SubType | Source | Action |
|---------|--------|--------|
| `NoEmail` | Actual bounce | Immediate suppression |
| `Suppressed` | SES account suppression | Immediate suppression |
| `EmailValidationSuppressed` | AWS Auto Validation | Immediate suppression |

### Enabling AWS Auto Validation

Enable Auto Validation in the AWS Console:

1. Go to **Amazon SES** → **Account dashboard**
2. Navigate to **Suppression** section
3. Enable **Auto Validation**
4. Choose threshold: **Amazon SES managed** (recommended), **High**, or **Medium**

Or via CLI:
```bash
aws sesv2 put-account-suppression-attributes \
  --suppressed-reasons BOUNCE COMPLAINT \
  --region us-east-1
```

### Cost Considerations

| Feature | Cost |
|---------|------|
| Auto Validation | $0.01 per 1,000 validations |
| Suppressed sends | Still count toward daily quota |
| Suppressed sends | Still charged standard send fee |

**With SESMailEngine:** Invalid addresses are suppressed after the first validation. Future sends to that address are blocked by SESMailEngine's suppression check *before* reaching SES, so you only pay for validation once per bad address.

### Benefits of Using Both

| SESMailEngine | AWS Auto Validation |
|---------------|---------------------|
| Reactive (handles bounces after they occur) | Proactive (prevents bounces before they occur) |
| Suppression from actual bounces | Suppression from predicted invalidity |
| Template management, tracking, notifications | Address validation only |
| **One-time validation cost** (suppresses after first check) | Charges per validation attempt |

**Recommendation:** Enable AWS Auto Validation for proactive protection. SESMailEngine handles the bounce notifications automatically, suppresses invalid addresses after the first validation (saving future validation costs), and provides the complete email infrastructure (templates, tracking, status notifications) that Auto Validation doesn't include.
