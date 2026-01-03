# Bounce Rate Protection

SESEmailEngine includes automatic bounce rate monitoring to protect your SES reputation.

## Overview

Amazon SES monitors your bounce rate and can suspend your sending privileges if it exceeds acceptable thresholds (typically 5-10%). SESEmailEngine proactively checks your daily bounce rate before sending each email and blocks sending when the threshold is exceeded.

## How It Works

1. **Before each email send**, the system checks the current daily bounce rate
2. **If the rate exceeds the threshold**, the email is rejected with a `bounce_rate_exceeded` error
3. **Results are cached for 5 minutes** to optimize DynamoDB costs

## Configuration

### Bounce Rate Threshold

Set via CloudFormation parameter `BounceRateThreshold`:

| Parameter | Default | Range | Description |
|-----------|---------|-------|-------------|
| `BounceRateThreshold` | 5 | 1-10 | Maximum allowed bounce rate percentage |

Example: Setting `BounceRateThreshold=5` means sending stops when bounce rate exceeds 5%.

## Caching Behavior

### Why Caching?

Checking bounce rate requires two DynamoDB queries:
- Count daily emails sent (EmailTracking table)
- Count daily bounces (Suppression table)

Without caching, high-volume senders would incur significant DynamoDB costs:
- 10,000 emails/day × 2 queries = 20,000 queries/day

### Cache Configuration

| Setting | Value | Description |
|---------|-------|-------------|
| Cache TTL | 5 minutes | How long results are cached |
| Cache Scope | Per Lambda instance | Each warm Lambda shares cache |

### Cost Savings

| Scenario | Without Cache | With 5-min Cache | Savings |
|----------|---------------|------------------|---------|
| 10,000 emails/day | 20,000 queries | 576 queries | 97% |
| 100,000 emails/day | 200,000 queries | 576 queries | 99.7% |

### Trade-offs

**Benefit**: Significant cost reduction and lower latency

**Trade-off**: Up to 5 minutes delay in detecting bounce rate spikes

**Risk Assessment**:
- At 100 emails/minute, ~500 emails could be sent before protection kicks in
- SES evaluates bounce rates over 24 hours, not minutes
- 500 emails during a spike won't significantly impact your SES reputation
- Bounce rates typically change gradually, not in sudden spikes

## Bypass Cache

For critical scenarios, you can bypass the cache:

```python
from bounce_quota_service import BounceQuotaService

service = BounceQuotaService()

# Normal check (uses cache)
result = service.check_bounce_rate()

# Force fresh query (bypasses cache)
result = service.check_bounce_rate(bypass_cache=True)

# Clear cache manually
service.clear_cache()
```

## Error Handling

When bounce rate is exceeded:

### EventBridge Events
- Returns success (200) to consume the event
- Logs the failure with reason `bounce_rate_exceeded`
- Email is NOT sent
- **Tracking record created** with status `failed` and error message

### SQS Retry Events (Soft Bounce Retries)
- Returns success (200) to consume the message
- Email is NOT sent
- **Tracking record created** with status `failed` and error message containing "during retry"
- Response includes `wasRetry: true` to indicate this was a retry attempt

This ensures **no silent email loss** - every blocked email (whether from EventBridge or SQS retry) gets a tracking record that customers can query.

### Tracking Record

When an email is blocked due to bounce rate, a tracking record is created in DynamoDB:

```json
{
  "emailId": "email-xxx",
  "status": "failed",
  "errorMessage": "Bounce rate exceeded: 6.2%",
  "toEmail": "recipient@example.com",
  "templateName": "welcome",
  "timestamp": "2024-12-19T10:30:00Z"
}
```

For retry attempts blocked by bounce rate:

```json
{
  "emailId": "email-yyy",
  "status": "failed",
  "errorMessage": "Bounce rate exceeded during retry: 6.2%",
  "toEmail": "recipient@example.com",
  "templateName": "welcome",
  "originalEmailId": "email-xxx",
  "retryAttempt": 1,
  "timestamp": "2024-12-19T10:45:00Z"
}
```

You can query blocked emails using the `to-email-timestamp-index` GSI.

## Monitoring

Check CloudWatch Logs for bounce rate warnings:

```
[WARNING] Bounce rate exceeded: 6.2% (threshold: 5.0%)
```

Filter pattern for blocked emails:
```
"bounce_rate_exceeded"
```

## Best Practices

1. **Set conservative thresholds**: Start with 3-4% rather than 5%
2. **Monitor trends**: Watch for gradual increases before hitting threshold
3. **Clean your lists**: Regularly remove invalid addresses
4. **Use double opt-in**: Reduces invalid addresses from the start
5. **Review DLQ**: Check EventBridge DLQ for blocked emails

## Troubleshooting

### Emails Being Blocked

1. Check current bounce rate:
   ```bash
   aws dynamodb query \
     --table-name ${STACK_NAME}-Suppression \
     --index-name suppression-type-date-index \
     --key-condition-expression "suppressionType = :type" \
     --expression-attribute-values '{":type":{"S":"bounce"}}'
   ```

2. Review recent bounces in Suppression table
3. Check SES console for account-level bounce rate
4. Consider temporarily increasing threshold (not recommended long-term)

### Cache Issues

If you suspect stale cache data:
1. Wait 5 minutes for cache to expire
2. Or deploy a new Lambda version (cold start = fresh cache)
3. Or use `bypass_cache=True` for critical checks

## Architecture

```
┌─────────────────┐     ┌──────────────────┐
│  Email Request  │────▶│  Bounce Rate     │
│  (EventBridge)  │     │  Check           │
└─────────────────┘     └────────┬─────────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
              Cache Hit?                 Cache Miss
                    │                         │
                    ▼                         ▼
            ┌───────────────┐       ┌─────────────────┐
            │ Return Cached │       │ Query DynamoDB  │
            │ Result        │       │ - EmailTracking │
            └───────────────┘       │ - Suppression   │
                                    └────────┬────────┘
                                             │
                                             ▼
                                    ┌─────────────────┐
                                    │ Calculate Rate  │
                                    │ Update Cache    │
                                    └────────┬────────┘
                                             │
                              ┌──────────────┴──────────────┐
                              │                             │
                        Rate OK                      Rate Exceeded
                              │                             │
                              ▼                             ▼
                    ┌─────────────────┐           ┌─────────────────┐
                    │ Proceed with    │           │ Block Email     │
                    │ Email Send      │           │ Log Warning     │
                    └─────────────────┘           └─────────────────┘
```

## Soft Bounce Handling

SESEmailEngine handles soft bounces (temporary delivery failures) with a balanced approach that provides quick feedback while protecting against truly problematic addresses.

### Soft Bounce Retry Behavior

When a soft bounce occurs (e.g., mailbox full, temporary rejection):

| Event | Action |
|-------|--------|
| First soft bounce | Retry once after 15 minutes |
| Retry also soft bounces | Mark as "failed" (no suppression) |
| 15+ soft bounces in 30 days | Permanent suppression |

### Why Single Retry?

The previous approach (3 retries over 45 minutes) was too aggressive:
- A single failed email chain could permanently suppress an address
- Customers lost control over whether to retry
- 45-minute retry window was often too long for transactional emails

The new approach provides:
- **Faster feedback**: 15 minutes vs 45 minutes
- **Customer control**: "failed" status lets you decide to resend
- **Cross-campaign protection**: 15 bounces/month threshold catches truly bad addresses

### Understanding "failed" vs "suppressed"

| Status | Meaning | Can Resend? |
|--------|---------|-------------|
| `failed` | Retry exhausted, but address NOT suppressed | Yes - your choice |
| `bounced` | Hard bounce, address permanently suppressed | No - will be blocked |
| `soft_bounced` | Temporary failure, retry scheduled | Wait for retry result |

### Cross-Campaign Suppression

To protect against truly problematic addresses, the system tracks soft bounces across all campaigns within a 30-day rolling window:

| Soft Bounces (30 days) | Action |
|------------------------|--------|
| 1-14 | Normal retry behavior |
| 15+ | Permanent suppression |

This prevents:
- Repeatedly sending to addresses that consistently fail
- Wasting SES quota on problematic addresses
- Damaging your sender reputation over time

### Checking Soft Bounce History

Query soft bounces for an email address:

```bash
aws dynamodb query \
  --table-name ${STACK_NAME}-EmailTracking \
  --index-name to-email-timestamp-index \
  --key-condition-expression "toEmail = :email" \
  --filter-expression "#status = :status" \
  --expression-attribute-names '{"#status": "status"}' \
  --expression-attribute-values '{":email":{"S":"user@example.com"},":status":{"S":"soft_bounced"}}'
```

### Resending Failed Emails

When an email has `failed` status due to retry exhaustion:

1. Check the `errorMessage` field for the failure reason
2. Decide if resending is appropriate (e.g., wait longer for mailbox full)
3. Publish a new EventBridge event with the same or new email ID
4. The system will attempt delivery again

```python
# Example: Resend a failed email
events.put_events(
    Entries=[{
        'Source': 'my.application',
        'DetailType': 'Email Request',
        'EventBusName': 'my-email-engine-EmailBus',
        'Detail': json.dumps({
            'to': 'user@example.com',  # Same recipient
            'templateName': 'welcome',
            'templateData': {'userName': 'John'},
            # New emailId for tracking (or omit to auto-generate)
            'emailId': f'welcome-{user_id}-retry-{datetime.now().isoformat()}'
        })
    }]
)
```

### Soft Bounce Types

| Bounce SubType | Description | Retry? |
|----------------|-------------|--------|
| `MailboxFull` | Recipient's mailbox is full | Yes |
| `MessageTooLarge` | Email too large for recipient | Yes |
| `ContentRejected` | Spam filter rejection | Yes |
| `General` | Unspecified temporary failure | Yes |
| `NoEmail` | Address doesn't exist | No (hard bounce) |
| `Suppressed` | On SES suppression list | No (hard bounce) |
