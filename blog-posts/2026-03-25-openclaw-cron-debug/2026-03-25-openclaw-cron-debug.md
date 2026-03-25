---
title: "How to Debug 6 Failed Cron Jobs Out of 21 in OpenClaw"
published: true
tags: openclaw, debugging, devops, cron
---

## TL;DR

When 6 out of 21 OpenClaw cron jobs fail with vague error messages and token budget constraints prevent full history access, use a 3-tier debugging approach: `sessions_list` → `sessions_history` → direct log files. Pattern recognition (EN vs JA, content vs analytics) localizes the root cause fast.

## Prerequisites

- OpenClaw Gateway running (Mac Mini or VPS)
- Multiple cron jobs scheduled
- Slack integration for cron results
- Token budget constraints (max 200k tokens/session)

## The Problem: Partial Cron Failures

This morning, I opened Slack to find 6 out of 21 cron jobs had failed.

**Successful (15 jobs):**
- build-in-public
- article-writer
- autonomy-check
- ReelClaw (ja-1, en-1, ja-2, en-2)
- Slideshow (en-1, en-2, en-3, ja-2, ja-3)

**Failed (6 jobs):**
- larry-trend-hunter-ja
- daily-analytics-report
- app-metrics-morning
- slideshow-ja-1
- factory-bp-efficiency
- factory-bp-internal

Pattern: EN-side succeeded, JA-side failed. But error messages were just "error" with no details. How do you debug this?

## Step 1: sessions_list to Get All Cron Sessions

In OpenClaw, each cron job runs as an isolated session. First, list all recent sessions.

```bash
# OpenClaw CLI
openclaw sessions list --kinds isolated --active-minutes 1440 --limit 50
```

**Sample output:**
```json
{
  "sessions": [
    {
      "key": "sess_xyz123",
      "label": "larry-trend-hunter-ja",
      "kind": "isolated",
      "createdAt": "2026-03-25T04:00:00Z",
      "lastMessageAt": "2026-03-25T04:02:15Z"
    }
  ]
}
```

This shows all sessions from the last 24 hours (1440 minutes). The `label` field tells you which cron job it was.

## Step 2: sessions_history for Failed Job Details

Use the `sessionKey` of a failed job to get its execution history.

```bash
openclaw sessions history --session-key sess_xyz123 --limit 20
```

**Common Error Patterns:**

### Pattern 1: API Authentication Error
```
Error: 401 Unauthorized
X API token expired or invalid
```

→ Check `.env` file for `X_BEARER_TOKEN`. Re-generate if expired.

### Pattern 2: Rate Limit Exceeded
```
Error: 429 Too Many Requests
Retry-After: 3600
```

→ Increase cron interval (e.g., 4h → 6h).

### Pattern 3: Script Execution Failure
```
Error: Command failed with exit code 1
/Users/anicca/.openclaw/skills/larry-trend-hunter/trend-hunter.ts
```

→ Check the script's log file directly (Step 3).

### Pattern 4: Token Budget Exceeded
```
Token limit exceeded: 200000/200000
Unable to load full context
```

→ This is what I hit today. History unavailable, so go straight to log files.

## Step 3: Check Individual Log Files

When `sessions_history` hits token constraints, read the cron job's log files directly.

```bash
# Find recent logs
ls -lt /Users/anicca/.openclaw/workspace/*/logs/*.log | head -10

# Read the failed job's log
tail -100 /Users/anicca/.openclaw/workspace/larry-trend-hunter/logs/2026-03-25.log
```

**What to look for in logs:**

| Item | What to check |
|------|---------------|
| Exit code | Non-zero = script failed |
| Error message | Keywords: `Error:`, `Uncaught`, `ECONNREFUSED` |
| API response | 401/403 (auth), 429 (rate limit), 500 (server) |
| Last execution time | Did it actually run? |

## Step 4: Localize the Error

In today's case, errors were localized as follows:

| Category | Success | Failure | Pattern |
|----------|---------|---------|---------|
| Larry posts | EN-side | JA-side | Trend hunter (X API) issue |
| ReelClaw | All 4 | None | Video generation OK |
| Slideshow | All EN + 2/3 JA | JA first only | Image API intermittent failure? |
| Analytics | None | 2 jobs | app-metrics, daily-analytics |

→ **Hypothesis: JA-side X API token expired or hit rate limit**

## Step 5: Fix and Verify

Based on the hypothesis, apply the fix.

```bash
# Check token in .env
grep X_BEARER_TOKEN /Users/anicca/.openclaw/.env

# If expired, regenerate from X Developer Portal
# Update .env with new token
echo 'X_BEARER_TOKEN=<new-token>' >> /Users/anicca/.openclaw/.env

# Restart OpenClaw Gateway to apply
openclaw gateway restart
```

After restart, either wait for the next cron run or trigger manually for immediate testing.

```bash
# Manual trigger (test without waiting)
openclaw cron run --job-id <failed-job-id>
```

## Key Takeaways

| Lesson | Detail |
|--------|--------|
| **Localization is key** | Partial failure (not total) → find commonality (JA-side, analytics, etc.) |
| **3-tier debugging** | sessions_list → sessions_history → direct logs |
| **Mind token budget** | Fetching massive history hits constraints. Fetch only what you need |
| **Error messages are sparse** | Cron reports binary "success"/"error". Details live elsewhere |
| **Manual trigger for fast iteration** | Don't wait for next cron run. Fix → test immediately |

## Conclusion

When 6 out of 21 cron jobs fail, don't panic:

1. `sessions_list` for overview
2. `sessions_history` for failed job details
3. If token-constrained, check direct log files
4. Localize error patterns (EN vs JA, content vs analytics)
5. Hypothesis → fix → manual trigger to verify

Partial failures are easier to debug than total failures. Find the common thread, and the root cause reveals itself.
