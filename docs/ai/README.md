# AI Agent Reference Documentation

This folder contains structured documentation optimized for AI support agents.

## Purpose

These documents are designed to be included in AI agent context when helping users troubleshoot SESMailEngine issues. They complement the user-facing docs in `docs/` with machine-parseable formats and decision trees.

## Files

| File | Purpose | When to Use |
|------|---------|-------------|
| `ERROR_INDEX.md` | Error code catalog with regex patterns | Match user error messages |
| `DIAGNOSTIC_QUERIES.md` | AWS CLI commands for investigation | Query customer's AWS resources |
| `SUPPORT_GUIDE.md` | Decision trees, "how do I" answers, response templates | Guide diagnosis flow |
| `FEATURE_GUIDE.md` | Feature explanations, use cases, system behavior | Answer "how does X work?" questions |

## Usage

### For AI Agent Prompts

Include these files in your AI agent's context:

```
You are a support agent for SESMailEngine. You have access to:

1. docs/ai/ERROR_INDEX.md - Error patterns and codes
2. docs/ai/DIAGNOSTIC_QUERIES.md - AWS CLI diagnostic queries  
3. docs/ai/SUPPORT_GUIDE.md - Diagnosis decision trees and "how do I" answers
4. docs/ai/FEATURE_GUIDE.md - Feature explanations and use cases

When a user reports an issue:
1. Match their error against ERROR_INDEX.md
2. Follow the decision tree in SUPPORT_GUIDE.md
3. Provide diagnostic queries from DIAGNOSTIC_QUERIES.md
4. Link to user-facing docs for detailed explanations

When a user asks "how do I...":
1. Check SUPPORT_GUIDE.md for common "how do I" questions
2. Check FEATURE_GUIDE.md for detailed feature explanations
3. Provide step-by-step instructions with example commands
```

### Token Budget

| File | Lines | Chars | ~Tokens |
|------|-------|-------|---------|
| ERROR_INDEX.md | ~250 | ~8K | ~2K |
| DIAGNOSTIC_QUERIES.md | ~280 | ~9K | ~2.5K |
| SUPPORT_GUIDE.md | ~350 | ~12K | ~3K |
| FEATURE_GUIDE.md | ~400 | ~14K | ~3.5K |
| **Total** | ~1280 | ~43K | ~11K |

All four files fit in most LLM context windows (fits easily in 16K+ contexts).

## Maintenance

When adding new features:
1. Add error patterns to `ERROR_INDEX.md`
2. Add diagnostic queries to `DIAGNOSTIC_QUERIES.md`
3. Update decision trees in `SUPPORT_GUIDE.md`
4. Keep user-facing docs in `docs/` updated separately

## Not for Users

These files are internal reference for AI agents. Users should be directed to:
- `docs/TROUBLESHOOTING.md` for error explanations
- `docs/INTEGRATION.md` for usage examples
- `docs/SETUP.md` for deployment help
