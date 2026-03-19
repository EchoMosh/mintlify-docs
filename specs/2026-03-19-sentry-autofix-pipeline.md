# Sentry → Claude Code Auto-Fix Pipeline

**Date**: 2026-03-19
**Status**: Researched, not yet implemented
**Priority**: After public API

## Architecture

```
Sentry error → webhook POST → Inngest function →
  git worktree (isolation) → Claude Code CLI (headless) →
  Ralph Loop (fix/test/verify x3) → PR → human approval → merge
```

## Why Inngest (not GitHub Actions)

Already in our stack, durable execution, rate limiting built-in, follows same pattern as our Composio webhook router.

## Key Components

1. **Sentry Internal Integration** — webhook on new issue, sends to `/api/sentry/webhook`
2. **Webhook handler** — HMAC verification, error-type allowlist, emits Inngest event
3. **Inngest function** — creates worktree, runs Claude Code CLI, Ralph Loop (3 attempts), creates PR
4. **Safety guardrails** — scoped tools, worktree isolation, PR-only, rate limit 1/issue/hour, 5-min timeout

## Auto-Fixable Error Types

- TypeError, ReferenceError
- Cannot read property
- is not a function / is not defined
- Unhandled Promise Rejection

## Tools Available

- `@sentry/mcp-server-sentry` — official Sentry MCP server for rich context
- `claude -p "..." --allowedTools "..." --max-turns 25 --output-format json`
- Sentry Autofix (Seer) — built-in alternative worth evaluating first

## Estimated Effort

~2-3 days total implementation

## Full Research

See agent output for complete code snippets (webhook handler, Inngest function, GitHub Action alternative).
