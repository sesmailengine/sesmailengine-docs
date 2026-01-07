# AI-Assisted Troubleshooting

Get help from AI assistants (ChatGPT, Claude, Copilot, etc.) using these docs optimized for LLMs.

## Quick Start

1. Copy the content from the relevant file below
2. Paste it into your AI chat
3. Describe your problem

The AI will use our documentation to give you accurate, SESMailEngine-specific answers.

## Files

| File | Use When |
|------|----------|
| `ERROR_INDEX.md` | You're getting an error message |
| `SUPPORT_GUIDE.md` | Emails aren't sending or delivering |
| `DIAGNOSTIC_QUERIES.md` | You need AWS CLI commands to investigate |
| `FEATURE_GUIDE.md` | You want to understand how something works |

## Example

```
I'm using SESMailEngine and getting this error:
"Email suppressed: hard-bounce"

[Paste ERROR_INDEX.md content here]

What does this mean and how do I fix it?
```

## Tips

- **Include the exact error message** - Copy the full text, not a summary
- **Mention your stack name** - Replace `{STACK}` in commands with your actual name
- **All files fit in one chat** - Total ~11K tokens, works with most AI models

## See Also

- `docs/TROUBLESHOOTING.md` - Human-readable troubleshooting
- `docs/INTEGRATION.md` - Code examples and event formats
