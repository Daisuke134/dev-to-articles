---
title: "How to Debug OpenClaw Cron Jobs When Execution Succeeds But Slack Reports Fail"
published: true
tags: openclaw, cron, debugging, slack
---

## TL;DR
Running 22 OpenClaw cron jobs, 19 failed with "Message failed" — but **execution was 100% successful**. Mistaking reporting layer failures for execution failures can lead to breaking working systems. This article shows how to diagnose layered errors and avoid wrong fixes.

## Prerequisites
- OpenClaw Gateway v1.x
- Mac Mini (Apple Silicon) or VPS
- Slack channel-based cron reporting
- 20+ cron jobs running periodically

## Symptom: 19 Out of 22 Jobs Show "Message failed"

On March 14th, 2026, morning logs showed mass failures:

```
[ERROR] autonomy-check - Message failed
[ERROR] trend-hunter-5am - Message failed
[ERROR] larry-post-morning-en - Message failed
[ERROR] larry-post-morning-ja - Message failed
...
```

**Success rate: 2/22 (9.1%)**

First impression: "Total system failure." Reality: entirely different.

## Investigation: Execution Layer Was Healthy

### Step 1: Check Output Files

Verified Larry content generation jobs (10 posts) directories:

```bash
ls -la ~/.openclaw/workspace/larry/
```

**Result: All 10 directories exist, each with 6 generated slide images.**

### Step 2: Check Postiz API Posting History

Postiz dashboard → Analytics:

- 2026-03-14: **10 new posts**
- All delivered to TikTok EN/JA accounts
- Engagement tracking started

**Conclusion: Execution layer was 100% successful.**

### Step 3: Find Common Patterns in Errors

Failed 19 jobs shared:
- All had `delivery.mode = "announce"` (Slack reporting)
- All used same Slack channel (#metrics C091G3PKHL2)
- Time distribution varied (07:30, 08:00, 16:30, 17:00, 21:00...)

Successful 2 jobs (build-in-public, article-writer):
- Both in 23:00 hour (23:10, 23:30)
- Same Slack channel (#metrics)
- Same delivery.mode "announce"

## Root Cause: Intermittent Slack Messaging Layer Failure

Three hypotheses:

| Hypothesis | Evidence | Probability |
|------------|----------|-------------|
| Slack API rate limit | Burst posting (6 posts in 4 hours) | 60% |
| OAuth token expiry | Only 23:00 hour succeeded (auto-refresh?) | 30% |
| Network intermittence | Mac Mini → Slack connection unstable | 10% |

**Verification:**

```bash
# Check Slack API responses in Gateway log
tail -500 ~/.openclaw/logs/gateway.log | grep -A5 "slack"

# Compare Slack posting intervals between success/failure
cat ~/.openclaw/logs/gateway.log | grep "Message failed" | awk '{print $1, $2}'
```

## Fix Procedure

### ❌ Wrong Fixes (Don't Do This)

1. **Touch execution layer** → It's healthy; you'll break it
2. **Change cron schedules** → Problem isn't timing, it's Slack API
3. **Re-run all crons** → Creates duplicate posts

### ✅ Correct Fix Procedure

#### Step 1: Check Slack Token Status

Slack workspace settings → Apps → OpenClaw → OAuth & Permissions:

- Check token expiry date
- Verify scopes include `chat:write`, `chat:write.public`

#### Step 2: Restart Gateway (Reload Token)

```bash
openclaw gateway restart
```

#### Step 3: Test Posting

```bash
openclaw message send --channel slack --target 'C091G3PKHL2' \
  --message "Test: Slack delivery test from Anicca"
```

If successful, next cron run will auto-recover.

#### Step 4: Long-term Fix - Add Retry Mechanism

Add to OpenClaw config (`~/.openclaw/config.json`):

```json
{
  "cron": {
    "delivery": {
      "retryOnFailure": true,
      "maxRetries": 3,
      "retryDelayMs": 5000
    }
  }
}
```

## Key Takeaways

| Lesson | Detail |
|--------|--------|
| **Diagnose errors by layer** | "Message failed" ≠ execution failure. It's reporting layer only. |
| **Check output files first** | Cron success is determined by artifacts, not error logs. |
| **Don't touch healthy layers** | If execution succeeded, don't modify execution code. |
| **Find patterns in delivery success/failure** | Look for commonalities in time, channel, volume. |

## Summary

OpenClaw's "Message failed" **does not mean execution failure**. Because execution and reporting layers are separated, judging success solely from error logs can lead to breaking healthy systems.

**Debugging priority:**

1. **Check output files** (determine actual execution success)
2. **Extract error commonalities** (identify failure pattern)
3. **Exclude healthy layers** (limit fix scope)
4. **Fix isolated layer only** (Slack token reload)

This approach diagnosed 19 errors without touching any execution code.
