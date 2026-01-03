# Email Templates Guide

SESMailEngine uses Jinja2 templates stored in S3 for email content. This guide covers template structure, variables, and best practices.

## Template Structure

Each template is a folder in your S3 bucket containing three files:

```
templates/
└── welcome/
    ├── template.html      # HTML version
    ├── template.txt       # Plain text version
    └── metadata.json      # Configuration
```

## Template Files

### template.html

The HTML version of your email with Jinja2 variables:

```html
<!DOCTYPE html>
<html>
<head>
    <title>{{ subject }}</title>
</head>
<body>
    <h1>Welcome {{ userName }}!</h1>
    <p>Thanks for joining {{ companyName }}.</p>
</body>
</html>
```

### template.txt

Plain text fallback for email clients that don't support HTML:

```
Welcome {{ userName }}!

Thanks for joining {{ companyName }}.
```

### metadata.json

Template configuration:

```json
{
  "subject": "Welcome to {{ companyName }}!",
  "senderName": "{{ companyName }} Team",
  "senderEmail": "noreply@{{ domain }}",
  "replyTo": "support@{{ domain }}",
  "requiredVariables": ["userName", "companyName", "domain"],
  "optionalVariables": ["customMessage"],
  "version": "1.0.0"
}
```

**Metadata Fields:**

| Field | Required | Description |
|-------|----------|-------------|
| `subject` | Yes | Email subject line (supports Jinja2 variables) |
| `senderName` | No | Display name for sender (supports Jinja2 variables) |
| `senderEmail` | No | Sender email address (supports Jinja2 variables) |
| `replyTo` | No | Reply-To address (supports Jinja2 variables) - where replies go when recipient clicks "Reply" |
| `requiredVariables` | No | List of variables that must be provided in templateData |
| `optionalVariables` | No | List of optional variables (for documentation) |
| `version` | No | Template version for tracking customizations |

**Reply-To Example:**

For contact form notifications, use `replyTo` so replies go directly to the form submitter:

```json
{
  "subject": "Contact Form: {{ formSenderName }}",
  "senderName": "Contact Form",
  "senderEmail": "noreply@{{ domain }}",
  "replyTo": "{{ formSenderEmail }}",
  "requiredVariables": ["formSenderName", "formSenderEmail", "message"],
  "version": "1.0.0"
}
```

## Limits

| Limit | Value |
|-------|-------|
| Max file size (per file) | 256 KB |
| Max cached templates | 50 |
| Cache TTL | 5 minutes |

**Note:** If your template exceeds 256 KB, consider:
- Moving images to a CDN and using URLs instead of inline content
- Simplifying CSS (avoid embedding entire frameworks)
- Splitting into multiple templates

## Variables

Use Jinja2 syntax for dynamic content:

| Syntax | Description |
|--------|-------------|
| `{{ variable }}` | Output a variable |
| `{% if condition %}` | Conditional block |
| `{% for item in list %}` | Loop |

### Example with Conditionals

```html
{% if isPremium %}
    <p>Thank you for being a premium member!</p>
{% else %}
    <p>Upgrade to premium for more features.</p>
{% endif %}
```

### Example with Loops

```html
<ul>
{% for item in orderItems %}
    <li>{{ item.name }} - ${{ item.price }}</li>
{% endfor %}
</ul>
```

## Sending Emails with Template Data

When sending an email via EventBridge, provide template variables in `templateData`:

```json
{
  "source": "my.application",
  "detail-type": "Email Request",
  "detail": {
    "to": "user@example.com",
    "templateName": "welcome",
    "templateData": {
      "userName": "John",
      "companyName": "Acme Corp",
      "domain": "acme.com"
    }
  }
}
```

## Sender Email Priority

The sender email is resolved in this order:

1. `senderOverride.email` in the EventBridge event (highest priority)
2. `senderEmail` in template metadata.json
3. `DefaultSenderEmail` CloudFormation parameter (fallback)

**Important:** If no sender email can be resolved from any of these sources, the email will fail with error: `No sender email configured`.

### Sender Override Example

To override the sender for a specific email:

```json
{
  "source": "my.application",
  "detail-type": "Email Request",
  "detail": {
    "to": "user@example.com",
    "templateName": "welcome",
    "templateData": { "userName": "John" },
    "senderOverride": {
      "email": "custom@yourdomain.com",
      "name": "Custom Sender Name"
    }
  }
}
```

## Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `Template 'X' not found` | Template folder doesn't exist in S3 | Upload template to `templates/{name}/` |
| `Template 'X' missing template.html` | HTML file missing | Add `template.html` to template folder |
| `Template 'X' missing template.txt` | Text file missing | Add `template.txt` to template folder |
| `Template 'X' missing metadata.json` | Metadata file missing | Add `metadata.json` to template folder |
| `Template 'X' has invalid metadata.json` | JSON syntax error | Fix JSON syntax in metadata.json |
| `No sender email configured` | No sender in event, template, or default | Configure sender in one of the 3 priority levels |
| `Undefined variable: X` | Template uses variable not in templateData | Add missing variable to templateData |

## Starter Templates

SESMailEngine includes starter templates you can install after deployment. These are production-ready templates you can use as-is or customize.

### Available Starter Templates

| Template | Description | Required Variables |
|----------|-------------|-------------------|
| `welcome` | New user registration welcome email | `userName`, `companyName` |
| `password-reset` | Secure password reset with expiration | `userName`, `companyName`, `resetLink` |
| `purchase-confirmation` | Order/purchase confirmation receipt | `customerName`, `orderId`, `productName`, `price`, `companyName` |
| `email-verification` | Email address verification | `userName`, `verificationUrl`, `companyName` |
| `invoice` | Professional invoice with line items | `customerName`, `invoiceNumber`, `invoiceDate`, `items`, `total`, `companyName` |
| `subscription-renewal` | Subscription renewal reminder | `userName`, `planName`, `renewalDate`, `amount`, `companyName` |
| `contact-form-notification` | Internal contact form notification | `formSenderName`, `formSenderEmail`, `message` |
| `shipping-notification` | Order shipped with tracking | `customerName`, `orderId`, `trackingNumber`, `carrier`, `companyName` |

All starter templates support these common optional variables:
- `logoUrl` - Company logo for branding. See [Images and Logos](#images-and-logos) for details.
- `senderDisplayName` - Custom "From" name (defaults to `companyName`). Example: "Acme Team" or "John from Acme"

All templates feature:
- **Responsive design** - Mobile-optimized with 620px breakpoint
- **Outlook compatibility** - VML buttons for Outlook rendering
- **Graceful degradation** - Optional variables don't break templates
- **Plain text fallback** - Every template includes template.txt

### Installing Starter Templates

After deploying SESMailEngine, install starter templates using the Template Seeder Lambda.

**Option 1: AWS CLI (Recommended)**

Copy the command from CloudFormation outputs:

```bash
# Get the install command
aws cloudformation describe-stacks \
  --stack-name my-email-engine \
  --query 'Stacks[0].Outputs[?OutputKey==`InstallTemplatesCommand`].OutputValue' \
  --output text

# Run the command (example output)
aws lambda invoke --function-name my-email-engine-TemplateSeeder /dev/stdout
```

**Option 2: AWS Console**

1. Go to Lambda → Functions → `{stack-name}-TemplateSeeder`
2. Click the "Test" tab
3. Click "Test" button (no input needed)
4. Check the response for installed templates

### Starter Template Details

#### welcome

Welcome email for new user registration.

**Required Variables:**

| Variable | Description |
|----------|-------------|
| `userName` | Recipient's name displayed in the greeting ("Welcome, John! 👋") |
| `companyName` | Your company name - appears in header, intro text, and footer copyright |

**Optional Variables:**

| Variable | Description | Default/Behavior |
|----------|-------------|------------------|
| `logoUrl` | URL to your company logo image (publicly accessible) | If omitted, no logo shown |
| `companyUrl` | Link for company name in header | If omitted, company name is plain text (not clickable) |
| `ctaUrl` | URL for the call-to-action button | If omitted, entire button section hidden |
| `ctaText` | Text displayed on the CTA button | Defaults to "Get Started" |
| `nextSteps` | Array of strings for onboarding steps | If omitted, "What's next?" section hidden |
| `helpUrl` | URL to your help center | If omitted, help text hidden |
| `unsubscribeLink` | URL for unsubscribe action | If omitted, unsubscribe link hidden |
| `privacyUrl` | URL to privacy policy | If omitted, privacy link hidden |
| `showEmoji` | Show emoji in heading (👋) | Defaults to `true`. Set to `false` for a more formal appearance |
| `senderDisplayName` | Custom "From" name in email header | Defaults to `companyName`. Example: "Acme Team" |
| `senderEmail` | Sender email address | Falls back to CloudFormation `DefaultSenderEmail` parameter |
| `year` | Copyright year in footer | Defaults to 2026 |

**Full Example:**
```json
{
  "templateName": "welcome",
  "templateData": {
    "userName": "John",
    "companyName": "Acme Corp",
    "logoUrl": "https://acme.com/images/logo.png",
    "companyUrl": "https://acme.com",
    "ctaUrl": "https://acme.com/dashboard",
    "ctaText": "Go to Dashboard",
    "nextSteps": ["Set up your workspace", "Invite your team", "Create your first project"],
    "helpUrl": "https://acme.com/help",
    "unsubscribeLink": "https://acme.com/unsubscribe",
    "privacyUrl": "https://acme.com/privacy",
    "year": 2026
  }
}
```

**Minimal Example (required only):**
```json
{
  "templateName": "welcome",
  "templateData": {
    "userName": "John",
    "companyName": "Acme Corp"
  }
}
```

#### password-reset

Secure password reset email with expiration warning and security information.

**Required Variables:**

| Variable | Description |
|----------|-------------|
| `userName` | Recipient's name displayed in the greeting |
| `companyName` | Your company name - appears in header, body text, and footer |
| `resetLink` | Password reset URL - displayed as button and plain text fallback |

**Optional Variables:**

| Variable | Description | Default/Behavior |
|----------|-------------|------------------|
| `logoUrl` | URL to your company logo image | If omitted, no logo shown |
| `companyUrl` | Link for company name in header | If omitted, company name is plain text |
| `expiresIn` | Link expiration time text | Defaults to "24 hours" |
| `requestIp` | IP address of reset request | If omitted, not shown in request details |
| `requestTime` | Time of reset request | If omitted, not shown in request details |
| `securityTips` | Array of custom security tip strings | If omitted, shows 3 default tips: expiration time, password won't change until reset, and anti-phishing warning |
| `showEmoji` | Show emojis in heading (🔐) and security box (🔒) | Defaults to `true`. Set to `false` for a more formal appearance |
| `senderDisplayName` | Custom "From" name in email header | Defaults to `companyName`. Example: "Acme Security" |
| `senderEmail` | Sender email address | Falls back to CloudFormation `DefaultSenderEmail` parameter |
| `year` | Copyright year in footer | Defaults to 2026 |

**Full Example:**
```json
{
  "templateName": "password-reset",
  "templateData": {
    "userName": "John",
    "companyName": "Acme Corp",
    "resetLink": "https://acme.com/reset/xyz789",
    "logoUrl": "https://acme.com/images/logo.png",
    "companyUrl": "https://acme.com",
    "expiresIn": "1 hour",
    "requestIp": "192.168.1.1",
    "requestTime": "December 29, 2024 at 2:30 PM",
    "securityTips": [
      "This link will expire in 1 hour",
      "Your password will not change until you complete the reset",
      "Acme Corp will never ask for your password by email"
    ]
  }
}
```

**Minimal Example (required only):**
```json
{
  "templateName": "password-reset",
  "templateData": {
    "userName": "John",
    "companyName": "Acme Corp",
    "resetLink": "https://acme.com/reset/xyz789"
  }
}
```

#### purchase-confirmation

Order/purchase confirmation receipt.

**Required Variables:**

| Variable | Description |
|----------|-------------|
| `customerName` | Customer's name displayed in the greeting |
| `orderId` | Order/transaction ID shown in order summary |
| `productName` | Name of purchased product |
| `price` | Product price (formatted string, e.g., "$49.99") - shown as line item and total |
| `companyName` | Your company name - appears in header and footer |

**Optional Variables:**

| Variable | Description | Default/Behavior |
|----------|-------------|------------------|
| `logoUrl` | URL to your company logo image | If omitted, no logo shown |
| `companyUrl` | Link for company name in header | If omitted, company name is plain text |
| `productDescription` | Additional product details | If omitted, only product name shown |
| `downloadUrl` | Digital product download URL | If omitted, download button hidden |
| `supportEmail` | Support email address | If omitted, support contact hidden |
| `privacyUrl` | Privacy policy URL | If omitted, privacy link hidden |
| `termsUrl` | Terms of service URL | If omitted, terms link hidden |
| `showEmoji` | Show emoji in heading (🎉) | Defaults to `true`. Set to `false` for a more formal appearance |
| `senderDisplayName` | Custom "From" name in email header | Defaults to `companyName`. Example: "Acme Orders" |
| `senderEmail` | Sender email address | Falls back to CloudFormation `DefaultSenderEmail` parameter |
| `year` | Copyright year in footer | Defaults to 2026 |

**Full Example:**
```json
{
  "templateName": "purchase-confirmation",
  "templateData": {
    "customerName": "John",
    "orderId": "ORD-2024-1234",
    "productName": "Pro Plan Annual",
    "productDescription": "12-month subscription with all features",
    "price": "$199.00",
    "companyName": "Acme Corp",
    "logoUrl": "https://acme.com/images/logo.png",
    "companyUrl": "https://acme.com",
    "downloadUrl": "https://acme.com/download/abc123",
    "supportEmail": "support@acme.com",
    "privacyUrl": "https://acme.com/privacy",
    "termsUrl": "https://acme.com/terms"
  }
}
```

**Minimal Example (required only):**
```json
{
  "templateName": "purchase-confirmation",
  "templateData": {
    "customerName": "John",
    "orderId": "ORD-2024-1234",
    "productName": "Pro Plan Annual",
    "price": "$199.00",
    "companyName": "Acme Corp"
  }
}
```

#### email-verification

Email address verification for new accounts.

**Required Variables:**

| Variable | Description |
|----------|-------------|
| `userName` | Recipient's name displayed in the greeting |
| `verificationUrl` | Email verification URL - displayed as button and plain text fallback |
| `companyName` | Your company name - appears in header, body text, and footer |

**Optional Variables:**

| Variable | Description | Default/Behavior |
|----------|-------------|------------------|
| `logoUrl` | URL to your company logo image | If omitted, no logo shown |
| `companyUrl` | Link for company name in header | If omitted, company name is plain text |
| `verificationCode` | 6-digit verification code | If omitted, code box hidden (use URL-only flow) |
| `expiresIn` | Link expiration time text | If omitted, expiration notice hidden |
| `showEmoji` | Show emoji in heading (✉️) | Defaults to `true`. Set to `false` for a more formal appearance |
| `senderDisplayName` | Custom "From" name in email header | Defaults to `companyName`. Example: "Acme Accounts" |
| `senderEmail` | Sender email address | Falls back to CloudFormation `DefaultSenderEmail` parameter |
| `year` | Copyright year in footer | Defaults to 2026 |

**Full Example:**
```json
{
  "templateName": "email-verification",
  "templateData": {
    "userName": "John",
    "verificationUrl": "https://acme.com/verify/xyz789",
    "companyName": "Acme Corp",
    "logoUrl": "https://acme.com/images/logo.png",
    "companyUrl": "https://acme.com",
    "verificationCode": "847293",
    "expiresIn": "24 hours"
  }
}
```

**Minimal Example (required only):**
```json
{
  "templateName": "email-verification",
  "templateData": {
    "userName": "John",
    "verificationUrl": "https://acme.com/verify/xyz789",
    "companyName": "Acme Corp"
  }
}
```

#### invoice

Professional invoice with line items and totals.

**Required Variables:**

| Variable | Description |
|----------|-------------|
| `customerName` | Customer's name displayed in the greeting |
| `invoiceNumber` | Invoice ID (e.g., "INV-2024-001") - shown in heading and details |
| `invoiceDate` | Invoice date (formatted string, e.g., "December 29, 2024") |
| `items` | Array of line items (see format below) |
| `total` | Total amount (formatted string, e.g., "$53.90") |
| `companyName` | Your company name - appears in header and footer |

**Optional Variables:**

| Variable | Description | Default/Behavior |
|----------|-------------|------------------|
| `logoUrl` | URL to your company logo image | If omitted, no logo shown |
| `companyUrl` | Link for company name in header | If omitted, company name is plain text |
| `dueDate` | Payment due date | If omitted, due date row hidden |
| `subtotal` | Subtotal before tax | If omitted, subtotal row hidden |
| `tax` | Tax amount | If omitted, tax row hidden |
| `paymentUrl` | Online payment URL | If omitted, "Pay Now" button hidden |
| `pdfUrl` | PDF download URL | If omitted, PDF download link hidden |
| `notes` | Additional notes for the invoice | If omitted, notes section hidden |
| `supportEmail` | Billing support email | If omitted, support contact hidden |
| `showEmoji` | Show emoji in heading (📄) | Defaults to `true`. Set to `false` for a more formal appearance |
| `senderDisplayName` | Custom "From" name in email header | Defaults to `companyName`. Example: "Acme Billing" |
| `senderEmail` | Sender email address | Falls back to CloudFormation `DefaultSenderEmail` parameter |
| `year` | Copyright year in footer | Defaults to 2026 |

**Items Array Format:**

Each item in the `items` array should have:

| Field | Description | Required |
|-------|-------------|----------|
| `name` | Item name | Yes |
| `description` | Item description | No |
| `quantity` | Quantity | No (defaults to 1) |
| `amount` | Item amount (formatted string) | Yes |

```json
{
  "items": [
    {
      "name": "Pro Plan Subscription",
      "description": "Monthly subscription - January 2025",
      "quantity": 1,
      "amount": "$49.00"
    },
    {
      "name": "Additional Storage (10GB)",
      "quantity": 2,
      "amount": "$20.00"
    }
  ]
}
```

**Full Example:**
```json
{
  "templateName": "invoice",
  "templateData": {
    "customerName": "John Smith",
    "invoiceNumber": "INV-2024-001",
    "invoiceDate": "December 29, 2024",
    "dueDate": "January 28, 2025",
    "items": [
      {"name": "Pro Plan", "description": "Monthly subscription", "quantity": 1, "amount": "$49.00"},
      {"name": "Extra Storage", "quantity": 2, "amount": "$10.00"}
    ],
    "subtotal": "$59.00",
    "tax": "$5.90",
    "total": "$64.90",
    "companyName": "Acme Corp",
    "logoUrl": "https://acme.com/images/logo.png",
    "companyUrl": "https://acme.com",
    "paymentUrl": "https://acme.com/pay/INV-2024-001",
    "pdfUrl": "https://acme.com/invoices/INV-2024-001.pdf",
    "notes": "Thank you for your business!",
    "supportEmail": "billing@acme.com"
  }
}
```

**Minimal Example (required only):**
```json
{
  "templateName": "invoice",
  "templateData": {
    "customerName": "John Smith",
    "invoiceNumber": "INV-2024-001",
    "invoiceDate": "December 29, 2024",
    "items": [
      {"name": "Pro Plan", "amount": "$49.00"}
    ],
    "total": "$49.00",
    "companyName": "Acme Corp"
  }
}
```

#### subscription-renewal

Subscription renewal reminder.

**Required Variables:**

| Variable | Description |
|----------|-------------|
| `userName` | Recipient's name displayed in the greeting |
| `planName` | Subscription plan name (e.g., "Pro Plan") |
| `renewalDate` | Renewal date (formatted string, e.g., "January 15, 2025") |
| `amount` | Renewal amount (e.g., "$49.00/month") |
| `companyName` | Your company name - appears in header and footer |

**Optional Variables:**

| Variable | Description | Default/Behavior |
|----------|-------------|------------------|
| `logoUrl` | URL to your company logo image | If omitted, no logo shown |
| `companyUrl` | Link for company name in header | If omitted, company name is plain text |
| `renewUrl` | Subscription management URL | If omitted, "Manage Subscription" button hidden |
| `cancelUrl` | Cancellation URL | If omitted, cancel/modify link hidden |
| `unsubscribeLink` | Email unsubscribe URL | If omitted, unsubscribe link hidden |
| `privacyUrl` | Privacy policy URL | If omitted, privacy link hidden |
| `showEmoji` | Show emoji in heading (🔄) | Defaults to `true`. Set to `false` for a more formal appearance |
| `senderDisplayName` | Custom "From" name in email header | Defaults to `companyName`. Example: "Acme Subscriptions" |
| `senderEmail` | Sender email address | Falls back to CloudFormation `DefaultSenderEmail` parameter |
| `year` | Copyright year in footer | Defaults to 2026 |

**Full Example:**
```json
{
  "templateName": "subscription-renewal",
  "templateData": {
    "userName": "John",
    "planName": "Pro Plan",
    "renewalDate": "January 15, 2025",
    "amount": "$49.00/month",
    "companyName": "Acme Corp",
    "logoUrl": "https://acme.com/images/logo.png",
    "companyUrl": "https://acme.com",
    "renewUrl": "https://acme.com/account/subscription",
    "cancelUrl": "https://acme.com/account/cancel",
    "unsubscribeLink": "https://acme.com/unsubscribe",
    "privacyUrl": "https://acme.com/privacy"
  }
}
```

**Minimal Example (required only):**
```json
{
  "templateName": "subscription-renewal",
  "templateData": {
    "userName": "John",
    "planName": "Pro Plan",
    "renewalDate": "January 15, 2025",
    "amount": "$49.00/month",
    "companyName": "Acme Corp"
  }
}
```

#### contact-form-notification

Internal notification for website contact form submissions. Sent to your team when someone submits a contact form.

**Required Variables:**

| Variable | Description |
|----------|-------------|
| `formSenderName` | Name of person who submitted the form |
| `formSenderEmail` | Email of person who submitted the form - used for "Reply to" button |
| `message` | The message content (supports multiline text) |

**Optional Variables:**

| Variable | Description | Default/Behavior |
|----------|-------------|------------------|
| `logoUrl` | URL to your company logo image | If omitted, no logo shown |
| `companyUrl` | Link for company name in header | If omitted, company name is plain text |
| `companyName` | Your company name | Defaults to "Contact Form" |
| `subject` | Subject line from form (if provided) | If omitted, subject row hidden |
| `submittedAt` | Submission timestamp | If omitted, timestamp row hidden |
| `showEmoji` | Show emoji in heading (📬) | Defaults to `true`. Set to `false` for a more formal appearance |
| `senderDisplayName` | Custom "From" name in email header | Defaults to `companyName` or "Contact Form" |
| `senderEmail` | Sender email address | Falls back to CloudFormation `DefaultSenderEmail` parameter |

**Full Example:**
```json
{
  "templateName": "contact-form-notification",
  "templateData": {
    "formSenderName": "Jane Doe",
    "formSenderEmail": "jane@example.com",
    "message": "Hi, I'm interested in your Pro Plan. Can you tell me more about the features included?\n\nThanks!",
    "subject": "Question about pricing",
    "submittedAt": "December 29, 2024 at 2:30 PM",
    "companyName": "Acme Corp",
    "logoUrl": "https://acme.com/images/logo.png",
    "companyUrl": "https://acme.com"
  }
}
```

**Minimal Example (required only):**
```json
{
  "templateName": "contact-form-notification",
  "templateData": {
    "formSenderName": "Jane Doe",
    "formSenderEmail": "jane@example.com",
    "message": "Hi, I have a question about your services."
  }
}
```

**Note:** The "Reply to" button uses `mailto:{{ formSenderEmail }}`, so clicking it opens the user's email client with the sender's address pre-filled.

#### shipping-notification

Order shipped notification with tracking information.

**Required Variables:**

| Variable | Description |
|----------|-------------|
| `customerName` | Customer's name displayed in the greeting |
| `orderId` | Order ID (e.g., "ORD-2024-5678") |
| `trackingNumber` | Shipping tracking number |
| `carrier` | Shipping carrier name (e.g., "FedEx", "UPS", "USPS", "DHL") |
| `companyName` | Your company name - appears in header and footer |

**Optional Variables:**

| Variable | Description | Default/Behavior |
|----------|-------------|------------------|
| `logoUrl` | URL to your company logo image | If omitted, no logo shown |
| `companyUrl` | Link for company name in header | If omitted, company name is plain text |
| `trackingUrl` | Direct tracking URL | If omitted, "Track Your Package" button hidden |
| `estimatedDelivery` | Estimated delivery date | If omitted, delivery estimate row hidden |
| `items` | Array of shipped items (see format below) | If omitted, items section hidden |
| `shippingAddress` | Delivery address (multiline string, use `\n` for line breaks) | If omitted, address section hidden |
| `supportEmail` | Support email address | If omitted, support contact hidden |
| `showEmoji` | Show emoji in heading (📦) | Defaults to `true`. Set to `false` for a more formal appearance |
| `senderDisplayName` | Custom "From" name in email header | Defaults to `companyName`. Example: "Acme Shipping" |
| `senderEmail` | Sender email address | Falls back to CloudFormation `DefaultSenderEmail` parameter |
| `year` | Copyright year in footer | Defaults to 2026 |

**Items Array Format:**

Each item in the `items` array should have:

| Field | Description | Required |
|-------|-------------|----------|
| `name` | Item name | Yes |
| `quantity` | Quantity shipped | No (defaults to 1) |

```json
{
  "items": [
    {"name": "Wireless Headphones", "quantity": 1},
    {"name": "USB-C Cable", "quantity": 2}
  ]
}
```

**Full Example:**
```json
{
  "templateName": "shipping-notification",
  "templateData": {
    "customerName": "John",
    "orderId": "ORD-2024-5678",
    "trackingNumber": "794644790132",
    "carrier": "FedEx",
    "companyName": "Acme Corp",
    "logoUrl": "https://acme.com/images/logo.png",
    "companyUrl": "https://acme.com",
    "trackingUrl": "https://www.fedex.com/track?tracknumbers=794644790132",
    "estimatedDelivery": "January 2, 2025",
    "items": [
      {"name": "Wireless Headphones", "quantity": 1},
      {"name": "USB-C Cable", "quantity": 2}
    ],
    "shippingAddress": "John Smith\n123 Main St\nSan Francisco, CA 94102",
    "supportEmail": "support@acme.com"
  }
}
```

**Minimal Example (required only):**
```json
{
  "templateName": "shipping-notification",
  "templateData": {
    "customerName": "John",
    "orderId": "ORD-2024-5678",
    "trackingNumber": "794644790132",
    "carrier": "FedEx",
    "companyName": "Acme Corp"
  }
}
```

### Customizing Starter Templates

Starter templates are copied to your S3 bucket. To customize:

1. Download the template from S3
2. Edit the HTML/text/metadata files
3. **Bump the version** in `metadata.json` (e.g., `"version": "1.0.1"` → `"version": "1.1.0"`)
4. Upload back to S3

### Version-Based Updates

The Template Seeder uses semantic versioning to manage updates:

| Scenario | Action |
|----------|--------|
| Template doesn't exist | **Installed** (new) |
| Starter version > your version | **Updated** (you get new features) |
| Starter version = your version | **Skipped** (no changes needed) |
| Starter version < your version | **Skipped** (your customizations preserved) |

**How to preserve your customizations:**
- When you customize a template, bump its version number
- Example: Change `"version": "1.0.0"` to `"version": "1.1.0"`
- The seeder will never overwrite a template with a higher version

**Response format:**
```json
{
  "statusCode": 200,
  "installed": ["newsletter"],
  "updated": ["welcome"],
  "skipped": ["password-reset"],
  "message": "Installed 1, updated 1, skipped 1"
}
```

**Note:** Templates without a `version` field are treated as version `0.0.0` and will be updated.

---

## Images and Logos

The Template Bucket is **private** for security reasons. Email clients (Gmail, Outlook, etc.) cannot fetch images directly from private S3 buckets. Instead, pass image URLs via `templateData`.

### Recommended Approach: `logoUrl` Variable

Pass your logo URL when sending emails:

```json
{
  "templateName": "welcome",
  "templateData": {
    "userName": "John",
    "companyName": "Acme Corp",
    "logoUrl": "https://yourdomain.com/images/logo.png"
  }
}
```

In your template, use conditional rendering:

```html
{% if logoUrl %}
<img src="{{ logoUrl }}" alt="{{ companyName }}" style="max-height: 50px;">
{% endif %}
```

### Where to Host Images

| Option | Pros | Cons |
|--------|------|------|
| Your website | Simple, already public | Depends on website uptime |
| Public S3 bucket | Reliable, fast | Separate bucket needed |
| CDN (CloudFront, Cloudflare) | Fast, cached globally | Additional setup |
| Image hosting service | Easy setup | Third-party dependency |

### Why Not Store Images in Template Bucket?

The Template Bucket blocks public access (security best practice). When SES sends an email:
1. Lambda reads template from private S3 ✓
2. Email is sent to recipient ✓
3. Recipient's email client tries to load `<img src="s3://...">` ✗
4. S3 returns Access Denied (bucket is private) ✗

**Solution:** Host images on a publicly accessible URL and pass via `templateData`.

### Image Best Practices

- **Use HTTPS URLs** - Some email clients block HTTP images
- **Keep images small** - Under 100KB for faster loading
- **Use standard formats** - PNG, JPG, GIF (avoid WebP, SVG)
- **Include alt text** - For accessibility and when images don't load
- **Make logos optional** - Templates should work without images

---

## Best Practices

1. **Always include both HTML and text versions** - Some email clients only display plain text
2. **Keep templates under 50 KB** - Smaller templates load faster and render quicker
3. **Use URLs for images** - Don't embed base64 images; host them on a public URL
4. **Test with real data** - Verify templates render correctly before production use
5. **Version your templates** - Use the `version` field in metadata.json for tracking
6. **Make images optional** - Use `{% if logoUrl %}` so templates work without images

---

## Creating Custom Templates

You can create your own templates by uploading files to your S3 Template Bucket.

### Step 1: Create Template Folder Structure

Create a folder in S3 with your template name:

```
templates/
└── my-custom-template/
    ├── template.html      # Required: HTML version
    ├── template.txt       # Required: Plain text version
    └── metadata.json      # Required: Configuration
```

### Step 2: Create template.html

Use Jinja2 syntax for variables. Include responsive design for mobile:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{ subject }}</title>
    <style type="text/css">
        /* Mobile styles */
        @media only screen and (max-width: 620px) {
            .email-container { width: 100% !important; }
            .mobile-padding { padding: 20px !important; }
            .mobile-button { display: block !important; width: 100% !important; }
        }
    </style>
</head>
<body style="margin: 0; padding: 0; font-family: Arial, sans-serif; background-color: #f4f4f5;">
    <table role="presentation" width="100%" cellspacing="0" cellpadding="0" border="0">
        <tr>
            <td align="center" style="padding: 40px 10px;">
                <table role="presentation" class="email-container" width="600" cellspacing="0" cellpadding="0" border="0" style="background-color: #ffffff; border-radius: 8px;">
                    
                    <!-- Header -->
                    <tr>
                        <td align="center" style="padding: 40px;">
                            {% if logoUrl %}
                            <img src="{{ logoUrl }}" alt="{{ companyName }}" style="max-height: 60px; max-width: 180px;">
                            {% endif %}
                            <h1>{{ companyName }}</h1>
                        </td>
                    </tr>
                    
                    <!-- Content -->
                    <tr>
                        <td class="mobile-padding" style="padding: 40px;">
                            <p>Hi {{ userName }},</p>
                            <p>{{ messageContent }}</p>
                            
                            {% if ctaUrl %}
                            <table role="presentation" width="100%">
                                <tr>
                                    <td align="center" style="padding: 20px 0;">
                                        <a href="{{ ctaUrl }}" class="mobile-button" style="display: inline-block; padding: 14px 32px; background-color: #3B5FE8; color: #ffffff; text-decoration: none; border-radius: 6px;">
                                            {{ ctaText | default('Click Here') }}
                                        </a>
                                    </td>
                                </tr>
                            </table>
                            {% endif %}
                        </td>
                    </tr>
                    
                    <!-- Footer -->
                    <tr>
                        <td style="padding: 30px; background-color: #f9fafb; text-align: center; font-size: 13px; color: #9ca3af;">
                            © {{ year | default(2026) }} {{ companyName }}
                        </td>
                    </tr>
                    
                </table>
            </td>
        </tr>
    </table>
</body>
</html>
```

### Step 3: Create template.txt

Plain text version for email clients that don't support HTML:

```
{{ companyName }}

Hi {{ userName }},

{{ messageContent }}

{% if ctaUrl %}
{{ ctaText | default('Click Here') }}: {{ ctaUrl }}
{% endif %}

---
(c) {{ year | default(2026) }} {{ companyName }}
```

### Step 4: Create metadata.json

Configure your template:

```json
{
  "subject": "{{ subject | default('Message from ' + companyName) }}",
  "senderName": "{{ companyName }}",
  "senderEmail": "noreply@{{ domain }}",
  "description": "My custom email template",
  "requiredVariables": ["userName", "companyName", "domain", "messageContent"],
  "optionalVariables": ["logoUrl", "ctaUrl", "ctaText", "subject", "year"],
  "version": "1.0.0"
}
```

### Step 5: Upload to S3

Using AWS CLI:

```bash
# Get your bucket name from CloudFormation outputs
BUCKET=$(aws cloudformation describe-stacks \
  --stack-name my-email-engine \
  --query 'Stacks[0].Outputs[?OutputKey==`TemplateBucketName`].OutputValue' \
  --output text)

# Upload template files
aws s3 cp template.html s3://$BUCKET/templates/my-custom-template/template.html
aws s3 cp template.txt s3://$BUCKET/templates/my-custom-template/template.txt
aws s3 cp metadata.json s3://$BUCKET/templates/my-custom-template/metadata.json
```

Or use the AWS Console to upload files to your bucket.

### Step 6: Test Your Template

Send a test email:

```json
{
  "source": "my.application",
  "detail-type": "Email Request",
  "detail": {
    "to": "test@example.com",
    "templateName": "my-custom-template",
    "templateData": {
      "userName": "Test User",
      "companyName": "My Company",
      "domain": "mycompany.com",
      "messageContent": "This is a test message.",
      "ctaUrl": "https://mycompany.com",
      "ctaText": "Visit Website"
    }
  }
}
```

### Jinja2 Quick Reference

| Syntax | Description | Example |
|--------|-------------|---------|
| `{{ var }}` | Output variable | `{{ userName }}` |
| `{{ var \| default('X') }}` | Default value | `{{ year \| default(2026) }}` |
| `{% if var %}...{% endif %}` | Conditional | `{% if logoUrl %}<img>{% endif %}` |
| `{% for item in list %}` | Loop | `{% for item in items %}...{% endfor %}` |
| `{{ var \| upper }}` | Uppercase | `{{ name \| upper }}` |
| `{{ var \| lower }}` | Lowercase | `{{ email \| lower }}` |

### Template Design Tips

1. **Use tables for layout** - Email clients have poor CSS support
2. **Inline all styles** - Many clients strip `<style>` tags
3. **Test on multiple clients** - Gmail, Outlook, Apple Mail render differently
4. **Keep width under 600px** - Standard email width
5. **Use web-safe fonts** - Arial, Helvetica, Georgia, Times New Roman
6. **Add alt text to images** - For accessibility and when images don't load
7. **Make buttons large** - At least 44px height for mobile tap targets
8. **Test responsive design** - Resize browser to 320px width
