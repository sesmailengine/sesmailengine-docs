# SESMailEngine Documentation

[![License](https://img.shields.io/badge/License-Commercial-blue.svg)](LICENSE)
[![AWS](https://img.shields.io/badge/AWS-Native-orange.svg)](https://aws.amazon.com/)
[![Documentation](https://img.shields.io/badge/Documentation-Complete-green.svg)](#documentation)

**Production-ready AWS email infrastructure that runs entirely in your account**

SESMailEngine is AWS-native email infrastructure that solves the scattered email logic problem. Instead of managing email code across multiple Lambda functions, get a centralized solution that scales to zero when idle and avoids recurring monthly contracts.

## What We Built This For

While building production SaaS applications on AWS, we faced a common problem: **email logic scattered across multiple Lambda functions**. We needed a centralized solution that:

- **Runs entirely in our AWS account** - Full control, no external dependencies
- **Scales to zero when idle** - Pay only when you send, not monthly fees
- **Avoids recurring contracts** - One-time purchase, unlimited projects

When no existing solution met these requirements, we built SESMailEngine.

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
- **Automatic retries** for soft bounces with intelligent backoff
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

**→ [Detailed Benchmark Report](docs/benchmarks/)** - Complete load test results and methodology
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
- **[Performance Benchmarks](docs/benchmarks/)** - Load test results and industry comparisons
## Architecture

**Production-grade email infrastructure with comprehensive monitoring and bounce protection**

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
│  │   Custom Event Bus  │  │    Event Rule       │  │   Dead Letter Queue │        │
│  │   Retry Policy      │  │    Pattern Match    │  │   Failed Events     │        │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘        │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
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
│  │   Bounce Handler    │  │   Status Updater    │  │   Retry Scheduler   │        │
│  │   Hard/Soft Logic   │  │   Tracking Records  │  │   Soft Bounce Queue │        │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘        │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
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