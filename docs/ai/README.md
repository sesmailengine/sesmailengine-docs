# AI-Assisted Troubleshooting

This folder contains documentation optimized for AI assistants (ChatGPT, Claude, Copilot, etc.) to help you troubleshoot SESMailEngine issues.

## How to Use

Point your AI assistant to these files when you need help:

```
Please read these SESMailEngine docs and help me troubleshoot:
- https://github.com/YOUR_ORG/sesmailengine/blob/main/docs/ai/ERROR_INDEX.md
- https://github.com/YOUR_ORG/sesmailengine/blob/main/docs/ai/SUPPORT_GUIDE.md
```

Or copy the file contents directly into your AI chat.

## Files

| File | Purpose | When to Use |
|------|---------|-------------|
| `ERROR_INDEX.md` | Error code catalog with causes and fixes | "I'm getting this error..." |
| `DIAGNOSTIC_QUERIES.md` | AWS CLI commands to investigate issues | "How do I check if..." |
| `SUPPORT_GUIDE.md` | Step-by-step troubleshooting guides | "My email isn't being delivered" |
| `FEATURE_GUIDE.md` | Feature explanations and how-to guides | "How do I..." |

## Example Prompts

### Troubleshooting an Error
```
I'm using SESMailEngine and getting this error:
"Template 'welcome' not found in bucket"

[Paste ERROR_INDEX.md content]

What's wrong and how do I fix it?
```

### Understanding a Feature
```
I want to set up different sender addresses for different email types in SESMailEngine.

[Paste FEATURE_GUIDE.md content]

How should I configure this?
```

### Diagnosing Delivery Issues
```
My emails aren't being delivered. The tracking table shows status "failed".

[Paste SUPPORT_GUIDE.md content]

Help me diagnose the issue.
```

## Token Budget

All files are designed to fit in most AI context windows:

| File | ~Tokens |
|------|---------|
| ERROR_INDEX.md | ~2K |
| DIAGNOSTIC_QUERIES.md | ~2.5K |
| SUPPORT_GUIDE.md | ~3K |
| FEATURE_GUIDE.md | ~3.5K |
| **Total** | ~11K |

You can include all four files in a single conversation with most AI models.

## Tips for Best Results

1. **Include the relevant file** - Don't just describe your problem; give the AI the reference docs
2. **Provide your stack name** - Replace `{STACK}` in commands with your actual stack name
3. **Share error messages** - Copy the exact error text, not a summary
4. **Include context** - What were you trying to do when the error occurred?

## Also See

- `docs/TROUBLESHOOTING.md` - Human-readable troubleshooting guide
- `docs/INTEGRATION.md` - Integration examples and event formats
- `docs/SETUP.md` - Deployment and configuration
