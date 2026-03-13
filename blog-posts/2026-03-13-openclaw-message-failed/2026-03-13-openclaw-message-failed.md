---
title: "How to Debug Message Failed Errors in OpenClaw Cron Jobs"
published: true
tags: openclaw, cron, slack, debugging
---

## TL;DR

When OpenClaw cron jobs fail with "Message failed" errors, the execution layer (Mac Mini + Gateway) is often healthy—the problem is in the delivery layer (Slack). This guide shows how to isolate and fix messaging failures.

## Prerequisites

- OpenClaw Gateway running
- Cron jobs configured
- Slack channel delivery enabled

## The Problem

```
Error: Message failed
```

Six cron jobs (autonomy-check, trend-hunter, app-metrics, app-resubmission-daily, factory-bp-revenue, factory-bp-efficiency) showed this error:

- Mac Mini infrastructure: healthy
- OpenClaw Gateway: running
- Some crons (build-in-public, article-writer): working fine

## Step 1: Identify the Error Layer

OpenClaw cron jobs operate in 3 layers:

| Layer | Role | How to Check |
|-------|------|--------------|
| Infrastructure | Mac Mini/VPS, OpenClaw Gateway | `openclaw gateway status` |
| Job Execution | Cron script execution | Check cron log files |
| Delivery Layer | Slack/messaging | `openclaw message send` test |

**Problem location:** Delivery layer. Execution itself was successful.

## Step 2: Verify Slack API Token

```bash
# Check if environment variable is set
echo $SLACK_BOT_TOKEN

# Verify correct format (starts with xoxb-)
# Invalid: empty, old token, wrong scope
```

## Step 3: Validate Channel ID

You must use the **channel ID**, not the channel name.

```bash
# Channel ID format: C091G3PKHL2
# How to find: Slack app → Channel → Right-click → View channel details → ID at bottom

# Wrong
openclaw message send --target '#metrics'  # NG: channel name

# Correct
openclaw message send --target 'C091G3PKHL2'  # OK: channel ID
```

## Step 4: Manual Test for Network Connectivity

```bash
# Minimal message send test
openclaw message send \
  --channel slack \
  --target 'C091G3PKHL2' \
  --message 'test'
```

**If successful:** Token and channel ID are correct → check cron job delivery settings
**If failed:** Token or channel ID is incorrect

## Step 5: Check Cron Job Delivery Settings

OpenClaw cron jobs specify delivery via the `delivery` field:

```json
{
  "delivery": {
    "mode": "announce",
    "channel": "slack",
    "to": "C091G3PKHL2"
  }
}
```

**Common mistakes:**

| Mistake | Correct Setting |
|---------|----------------|
| `"mode": "silent"` | `"mode": "none"` or `"announce"` |
| `"to": "#metrics"` | `"to": "C091G3PKHL2"` |
| Missing `delivery` field | Add it |

## Step 6: Reference Successful Patterns from History

```bash
# Check configuration of previously successful cron jobs
openclaw cron list --includeDisabled true | jq '.[] | select(.name == "build-in-public")'
```

Copy the `delivery` settings from working jobs and apply to failing ones.

## Step 7: Consider Retry Logic (Future Improvement)

OpenClaw has no message delivery retry mechanism. Consider:

1. **Cron-level retry**: Script retries `openclaw message send` 3 times
2. **Fallback on failure**: Write to local log file
3. **Monitoring alerts**: Notify alternative channel on delivery failure

## Key Takeaways

| Lesson | Detail |
|--------|--------|
| Layer isolation is critical | Infrastructure can be healthy while delivery fails |
| Use channel ID | Not channel name (#metrics) but ID (C091G3PKHL2) |
| Copy successful patterns | Reference working job configurations |
| Manual test with minimal config | Verify with `openclaw message send` |

**Diagnosis result:** System infrastructure is healthy. Fix delivery settings to resolve.
