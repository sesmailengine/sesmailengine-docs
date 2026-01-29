# SESMailEngine Documentation

[![License](https://img.shields.io/badge/License-Commercial-blue.svg)](LICENSE)
[![AWS](https://img.shields.io/badge/AWS-Native-orange.svg)](https://aws.amazon.com/)
[![Documentation](https://img.shields.io/badge/Documentation-Complete-green.svg)](#documentation)

**Production-ready AWS email infrastructure that runs entirely in your account**

🌐 **[Get SESMailEngine at sesmailengine.com](https://www.sesmailengine.com)** - Purchase, download, and get support

SESMailEngine is AWS-native email infrastructure that protects your SES reputation. By centralizing all email sending through one controlled path, it prevents the architecture mistakes that silently damage your sender reputation. Scales to zero when idle, no recurring monthly contracts.

## The SES Reputation Problem

**Amazon SES is cheap, powerful, and reliable — until your reputation drops.**

Most SES reputation damage doesn't come from spam campaigns. It comes from **architecture mistakes**: too many services sending email independently, no shared suppression logic, and no central visibility into bounces or complaints.

### Why This Happens

Most AWS architectures evolve like this:
- Service A sends password reset emails
- Service B sends invoices  
- Service C sends notifications
- Each has its own Lambda, each calls `SendEmail` directly

This feels fine at first. But now you have:
- ❌ No single place to enforce suppression
- ❌ No shared bounce-rate logic
- ❌ No consistent tracking
- ❌ No way to stop bad sends centrally

**SES reputation is account-wide.** One misbehaving Lambda, one bad import, one forgotten suppression check can silently damage everything that sends email from your account.

### The AI Agent Risk

A growing threat: **AI agents with direct SES access via tool calling.**

When you give an AI coding assistant or autonomous agent the ability to call AWS APIs directly, you're trusting it to never hallucinate an email address. But AI models do hallucinate — they generate plausible-looking but invalid email addresses, especially when:
- Inferring recipient addresses from context
- Auto-completing partially provided addresses
- Generating test data that accidentally gets sent

Each hallucinated email that bounces damages your SES reputation. Unlike human mistakes that happen occasionally, an AI agent can generate hundreds of bad sends in minutes before anyone notices.

**The fix:** Never give AI agents direct SES access. Route all email through a controlled path with suppression checks — exactly what SESMailEngine provides.

### What AWS Tells You

AWS recommends you build proper email infrastructure:

> *"Send only high-quality emails to recipients who expect to hear from you."*
> — [AWS SES Best Practices](https://docs.aws.amazon.com/ses/latest/dg/tips-and-best-practices.html)

> *"Set up a process to handle bounces and complaints."*
> — [Monitoring Sending Activity](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity.html)

### What SES Gives You vs. What It Doesn't

| SES Provides | SES Does NOT Provide |
|--------------|---------------------|
| Bounce & complaint events | Enforcement across services |
| Suppression lists | Central sending policies |
| Reputation metrics | Shared bounce-rate logic |
| Event publishing | Protection from architecture mistakes |

**AWS assumes you will build the control plane. Most teams don't.**

### The Core Principle

To protect SES reputation, you need **one central sending path**.

Not: *"Every service can send email"*

But: *"Every service requests an email to be sent"*

That single change unlocks everything.

## What We Built

We ran into this problem ourselves while operating multiple AWS workloads using SES. When no existing solution met our requirements, we built SESMailEngine — a centralised email infrastructure that actively protects your reputation.

**SESMailEngine implements the "one sender, many producers" pattern:**

- ✅ **Single sending path** - All emails flow through one controlled system
- ✅ **Shared suppression** - One suppression table protects all services
- ✅ **Central bounce handling** - Automatic hard/soft bounce logic
- ✅ **Cross-campaign protection** - Bad addresses blocked everywhere
- ✅ **Real-time monitoring** - 6 CloudWatch alarms catch problems early
- ✅ **Zero data loss** - 3 Dead Letter Queues capture every failure

### Why We Built It This Way

- **Runs entirely in your AWS account** - Full control, no external dependencies
- **Scales to zero when idle** - Pay only when you send, not monthly fees
- **Avoids recurring contracts** - One-time purchase, unlimited projects

## What We Are

- **Production-ready AWS email infrastructure** - Deploy in minutes with CloudFormation
- **One-time purchase, unlimited projects** - No per-seat or monthly subscription fees
- **Full control in your AWS account** - You own and control all email data
- **Zero idle costs** - Pay only AWS usage when you send emails
- **Simple EventBridge integration** - Publish events, we handle the rest

## What We're Not

- **Not a managed service** - You deploy and own the infrastructure
- **Not a marketing automation platform** - We focus on transactional email infrastructure
- **Not a drag-and-drop email builder** - Use your own templates or our starter templates
- **Not a per-seat subscription** - One license, unlimited team members
- **Not a black box** - You see everything, full CloudFormation source included

## Key Features

### 🛡️ Automatic Reputation Protection
- **Hard and soft bounce handling** with automatic suppression
- **Spam complaint management** with cross-campaign protection
- **Automatic sending pause** when bounce thresholds are exceeded
- **Cross-campaign protection** prevents repeat offenders

### 📊 Built-in Tracking & Analytics
- **Track every email in real time** - sent, delivered, bounced, complained, opened
- **Query by recipient, message ID, or date** using DynamoDB indexes
- **Full audit trail** with retry tracking and status transitions
- **Zero data loss architecture** - every email is tracked

### 🚨 6 Pre-Configured CloudWatch Alarms
- **Bounce rate monitoring** (>3% triggers alert, AWS suspends at ~5%)
- **Complaint rate monitoring** (>0.05% triggers alert, AWS suspends at ~0.1%)
- **Lambda error detection** for sustained issues
- **Dead Letter Queue monitoring** for failed processing
- **Admin notifications** enabled out of the box

### 🔒 No Silent Failures
- **Three Dead Letter Queues** capture every failure type
- **Soft bounce handling** - AWS SES retries automatically for up to 12 hours
- **All data encrypted at rest** across S3, DynamoDB, and SQS
- **Comprehensive error tracking** with detailed failure reasons

### 📧 Production-Ready Templates
- **Jinja2 templates** stored in S3 with versioning enabled
- **8 starter templates included** (welcome, password reset, invoices, etc.)
- **Template seeder Lambda** for one-command setup
- **Responsive design** with Outlook compatibility

### ⚡ Deploy in Minutes
- **Single CloudFormation template** with 10+ configuration options
- **Cross-platform Python installer** handles everything automatically
- **Production-ready infrastructure** in under 10 minutes
- **Zero external dependencies** - pure AWS services
## Perfect For

### 🚀 SaaS Applications
Onboard users, announce features, and handle billing notifications. EventBridge-triggered sending and Jinja2 templates let you scale effortlessly without increasing costs.

### 💳 Transactional Emails
Deliver critical communications like password resets, order confirmations, and shipping updates. Automatic bounce handling and reputation protection ensure high delivery rates.

### 🔔 Notification Systems
Send alerts, digests, and summaries from any AWS service. With CloudWatch alarms and Lambda integration, you can monitor delivery and errors in real time.

### 🛒 E-commerce
Email receipts, shipping notifications, and review requests at high volume. Suppression lists and bounce handling protect your sender reputation while keeping costs low.

### 🏢 Internal Tools
Keep your team informed with monitoring alerts, reports, and notifications. No per-seat pricing, full AWS-native infrastructure, and flexible templates make it ideal for internal communications.

### 📱 Mobile Apps
Handle account verifications, push fallback emails, and engagement campaigns. Dynamic templates, S3 versioning, and EventBridge triggers make mobile email reliable and scalable.
## Quick Start

### Python
```python
import boto3
import json

events = boto3.client('events')

response = events.put_events(
    Entries=[{
        'Source': 'my.application',
        'DetailType': 'Email Request',
        'EventBusName': 'sesmailengine-EmailBus',  # Your stack name
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
```

### TypeScript/Node.js
```typescript
import { EventBridgeClient, PutEventsCommand } from '@aws-sdk/client-eventbridge';

const client = new EventBridgeClient({});

const command = new PutEventsCommand({
  Entries: [{
    Source: 'my.application',
    DetailType: 'Email Request',
    EventBusName: 'sesmailengine-EmailBus',  // Your stack name
    Detail: JSON.stringify({
      to: 'recipient@example.com',
      templateName: 'welcome',
      templateData: {
        userName: 'John',
        companyName: 'My Company'
      }
    })
  }]
});

await client.send(command);
```

**→ [Complete Integration Guide](docs/INTEGRATION.md)** - Full examples for Python, TypeScript, Java, and more
## Performance

**Load tested with 10,000 emails:**

| Metric | Result |
|--------|--------|
| **Event Submission** | 493+ events/second |
| **Success Rate** | 100% (zero data loss) |
| **Processing Time** | 14.7 minutes total |
| **Data Integrity** | 0.0000% data loss rate |

### Industry Comparison

| Provider | Throughput | Data Loss | Cost (100k emails/month) |
|----------|------------|-----------|--------------------------|
| **SESMailEngine** | **493 events/sec** | **0.000%** | **~$3.90** |
| SendGrid | 10,000 events/sec | 0.100% | $20+ |
| Mailgun | 5,000 events/sec | 0.100% | $35+ |
| Postmark | 500 events/sec | 0.010% | $15+ |

**→ [Detailed Performance Information](docs/ARCHITECTURE.md#performance-characteristics)** - Complete throughput and cost analysis
## Documentation

### Getting Started
- **[Setup Guide](docs/SETUP.md)** - Deploy SESMailEngine to your AWS account in minutes
- **[Integration Guide](docs/INTEGRATION.md)** - Code examples for Python, TypeScript, Java, and more
- **[Templates Guide](docs/TEMPLATES.md)** - Create and customize email templates with Jinja2

### Technical Guides  
- **[Architecture Documentation](docs/ARCHITECTURE.md)** - Detailed system architecture and data flow diagrams
- **[Security & Compliance](docs/SECURITY.md)** - GDPR, encryption, IAM policies, and audit trails
- **[Bounce Rate Protection](docs/BOUNCE_RATE_PROTECTION.md)** - Automatic reputation management and monitoring

### Reference Materials
- **[Troubleshooting Guide](docs/TROUBLESHOOTING.md)** - Common issues, error codes, and solutions
- **[AI Documentation](docs/ai/)** - Structured docs optimized for AI coding assistants and LLMs
## Email Status Notification Relay

**Real-time status updates via EventBridge Fan-Out**

SESMailEngine publishes email status change events back to EventBridge, enabling your services to receive real-time notifications about delivery, bounces, complaints, and opens.

```
Your App → EventBridge → SESMailEngine → SES → Recipient
                ↑                           ↓
                └──── Status Notifications ←┘
```

### How It Works

When an email status changes, the Feedback Processor publishes an "Email Status Changed" event to the same EventBridge bus. Your services create rules to subscribe to these events.

### Available Status Events
- **delivered** - Email successfully delivered to recipient's mail server
- **bounced** - Permanent delivery failure (address doesn't exist)
- **soft_bounced** - Temporary failure (mailbox full, AWS SES retries for up to 12 hours)
- **complained** - Recipient marked email as spam
- **opened** - Recipient opened the email

### Subscribe to Status Updates

Create an EventBridge rule to receive notifications:

```bash
# Subscribe to bounce notifications for your service
aws events put-rule \
  --name "my-app-bounce-notifications" \
  --event-bus-name "sesmailengine-EmailBus" \
  --event-pattern '{
    "source": ["sesmailengine"],
    "detail-type": ["Email Status Changed"],
    "detail": {
      "status": ["bounced"],
      "originalSource": ["my.application"]
    }
  }'
```

### Use Cases
- **Update user records** when emails bounce permanently
- **Trigger workflows** when important emails are delivered
- **Track engagement** by monitoring email opens
- **Alert on complaints** to prevent reputation damage

**→ [Complete Status Notification Guide](docs/INTEGRATION.md#receiving-status-notifications)** - Full event schema, code examples, and best practices
## Architecture

**Production-grade email infrastructure with comprehensive monitoring, bounce protection, and real-time status notifications**

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              YOUR APPLICATION                                        │
│  aws events put-events --event-bus-name {StackName}-EmailBus                        │
│  --detail '{"to":"user@example.com", "templateName":"welcome", "templateData":{}}' │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                            AWS EVENTBRIDGE                                          │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐        │
│  │   Custom Event Bus  │  │    Event Rules      │  │   Dead Letter Queue │        │
│  │   Bidirectional     │  │    Send + Receive   │  │   Failed Events     │        │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘        │
└─────────────────────────────────────────────────────────────────────────────────────┘
                          │                                     ▲
                          ▼                                     │
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          EMAIL SENDER LAMBDA                                        │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐        │
│  │  Suppression Check  │  │  Template Rendering │  │   Bounce Rate Check │        │
│  │  Email Validation   │  │  Jinja2 Processing  │  │   Daily Quota Limit │        │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘        │
└─────────────────────────────────────────────────────────────────────────────────────┘
                    │                         │                         │
                    ▼                         ▼                         ▼
┌─────────────────────┐    ┌─────────────────────┐    ┌─────────────────────┐
│     DYNAMODB        │    │         S3          │    │        SES          │
│                     │    │                     │    │                     │
│  EmailTracking      │    │  Template Bucket    │    │  Configuration Set  │
│  ├── Status         │    │  ├── HTML Templates │    │  ├── TLS Required   │
│  ├── Timestamps     │    │  ├── Text Templates │    │  ├── Event Tracking │
│  ├── Retry History  │    │  └── Metadata       │    │  └── Reputation     │
│  └── 3 GSI Indexes  │    │                     │    │                     │
│                     │    │  Versioning Enabled │    │  Sends to Recipients│
│  Suppression Table  │    │  Encryption at Rest │    │  Publishes Events   │
│  ├── Hard Bounces   │    │                     │    │                     │
│  ├── Complaints     │    │                     │    │                     │
│  └── Soft Bounce    │    │                     │    │                     │
│      Exceeded       │    │                     │    │                     │
└─────────────────────┘    └─────────────────────┘    └─────────────────────┘
                                                                   │
                                                                   │ SES Events
                                                                   ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                 AWS SNS                                              │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐        │
│  │   SES Feedback      │  │   Event Routing     │  │   Dead Letter Queue │        │
│  │   bounce/complaint  │  │   delivery/open     │  │   Failed Processing │        │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘        │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        FEEDBACK PROCESSOR LAMBDA                                    │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐        │
│  │   Bounce Handler    │  │   Status Updater    │  │  Status Notifier    │        │
│  │   Hard/Soft Logic   │  │   Tracking Records  │  │  → EventBridge      │        │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘        │
└─────────────────────────────────────────────────────────────────────────────────────┘
                          │                                     │
                          ▼                                     │ "Email Status Changed"
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              AWS SQS RETRY QUEUE                                    │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐        │
│  │   Soft Bounce       │  │   15-min Delay      │  │   Dead Letter Queue │        │
│  │   Retry Messages    │  │   Single Retry      │  │   Failed Retries    │        │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘        │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        │ Triggers Email Sender Lambda
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           CLOUDWATCH ALARMS (6 ALARMS)                              │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐        │
│  │   Bounce Rate >3%   │  │   DLQ Depth > 0     │  │   Lambda Errors     │        │
│  │   Complaint >0.05%  │  │   All 3 DLQs        │  │   Sustained Issues  │        │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘        │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Key Benefits
- **Zero data loss architecture** - 3 Dead Letter Queues capture all failures
- **Intelligent bounce handling** - Automatic suppression with soft bounce retries
- **Comprehensive monitoring** - 6 CloudWatch alarms protect your reputation
- **Real-time status notifications** - EventBridge fan-out for delivery, bounce, and open events
- **Scales to zero** - Pay only when you send, serverless throughout
- **Runs in your AWS account** - Full control, complete audit trail

**→ [Complete Architecture Guide](docs/ARCHITECTURE.md)** - Detailed diagrams, data flow, and component interactions
## AI-Optimized Documentation

**Enhanced support for AI coding assistants and LLMs**

The `docs/ai/` directory contains structured documentation optimized for AI agents:

- **[ERROR_INDEX.md](docs/ai/ERROR_INDEX.md)** - Error patterns and diagnostic codes for automated troubleshooting
- **[DIAGNOSTIC_QUERIES.md](docs/ai/DIAGNOSTIC_QUERIES.md)** - AWS CLI commands for system investigation
- **[SUPPORT_GUIDE.md](docs/ai/SUPPORT_GUIDE.md)** - Decision trees and "how do I" answers for common questions
- **[FEATURE_GUIDE.md](docs/ai/FEATURE_GUIDE.md)** - Detailed feature explanations and use case examples

**Purpose:** These files provide machine-parseable formats and decision trees to help AI coding assistants provide better support when integrating SESMailEngine.

**→ [AI Documentation Overview](docs/ai/README.md)** - Complete guide for AI agent integration
## Security

### Enterprise-Grade Security
- **Encryption at rest and in transit** - All data encrypted using AWS-managed keys (TLS 1.2+)
- **Least-privilege IAM policies** - No wildcard permissions, minimal required access
- **Data isolation** - All processing occurs within your AWS account, no external calls
- **Audit trails** - Complete CloudTrail integration for compliance and monitoring

### Compliance Ready
- **GDPR & UK GDPR** - Configurable data retention (30-365 days) and right to erasure
- **SOC 1/2/3, ISO 27001** - Inherits AWS compliance certifications
- **No email content storage** - Only metadata retained for tracking purposes
- **Point-in-time recovery** - 35-day backup window for accidental deletion protection

**→ [Complete Security Guide](docs/SECURITY.md)** - Detailed compliance information (GDPR, SOC, ISO), encryption details, and audit procedures
## Cost Model

### One-Time Purchase
- **No monthly subscriptions** - Pay once, use forever across unlimited projects
- **No per-seat pricing** - Entire team can use with single license
- **No recurring contracts** - You own the infrastructure code permanently

### Zero Idle Costs
- **Pay only AWS usage when you send** - Serverless architecture scales to zero
- **Most usage falls within AWS Free Tier** - 62,000 emails/month free when sent from Lambda
- **Transparent pricing** - Only pay standard AWS rates (SES, Lambda, DynamoDB, S3)

### Cost Examples (AWS Free Tier)
| Monthly Volume | Total Cost |
|----------------|------------|
| 10,000 emails | ~$0.01 |
| 62,000 emails | ~$0.06 |
| 100,000 emails | ~$3.90 |

**vs. SaaS alternatives:** SendGrid ($20+), Mailgun ($35+), Postmark ($15+) for 100k emails/month

**→ [Detailed Cost Analysis](docs/ARCHITECTURE.md#cost-estimate-with-aws-free-tier)** - Complete breakdown with AWS Free Tier calculations
## Get SESMailEngine

**Ready to deploy production-ready email infrastructure?**

### 🛒 Purchase & Download
**[Visit sesmailengine.com](https://www.sesmailengine.com)** to purchase your license and download the latest version.

- One-time purchase, unlimited projects
- Instant download after purchase
- 30-day money-back guarantee
- Commercial support included

### 📚 Documentation Issues
Found an error in our documentation? Have a suggestion for improvement?

**[Report Documentation Issues](https://github.com/sesmailengine/sesmailengine-docs/issues)** - Help us improve these docs for everyone

### 💬 Support
- **Product support:** Available at [sesmailengine.com](https://www.sesmailengine.com)
- **Documentation issues:** Use GitHub Issues in this repository
- **Technical questions:** Check our [Troubleshooting Guide](docs/TROUBLESHOOTING.md) first

---

**SESMailEngine** - Production-ready AWS email infrastructure that runs entirely in your account.

*Deploy in minutes. Scale to zero. Pay only when you send.*