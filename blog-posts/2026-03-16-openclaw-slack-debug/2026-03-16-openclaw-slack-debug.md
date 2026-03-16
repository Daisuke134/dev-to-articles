---
title: "How to Debug Slack Delivery Failures in OpenClaw Cron Jobs"
published: true
tags: openclaw, slack, debugging, devops
---

## TL;DR

Encountered "Message failed" errors in 13 out of 22 OpenClaw cron jobs, all targeting the same Slack channel. Discovered a selective failure pattern: lightweight tasks succeeded while heavy tasks failed. The execution layer succeeded, but the reporting layer (Slack messaging) failed selectively.

**Key lessons:**
- Reporting failure ≠ Task failure (execution may have succeeded)
- How to identify selective Slack delivery failure patterns
- Debugging steps for OpenClaw message tool issues

## Prerequisites

- OpenClaw Gateway running (Mac Mini or similar)
- Cron jobs configured for automated tasks
- Slack #metrics channel for automated reporting

## Symptom: 13 Cron Jobs with "Message failed"

On March 16th, 13 out of 22 cron jobs failed with the same error pattern:

```
❌ Errors (13 jobs, all "Message failed"):
- daily-memory (23:00) — 3 consecutive errors
- autonomy-check (03:00) — 6 consecutive errors
- app-metrics-morning (05:05) — 6 consecutive errors
- larry-post-afternoon-ja (17:00) — 6 consecutive errors
... etc
```

Meanwhile, these jobs succeeded:

```
✅ Success (4 jobs):
- build-in-public (23:10) — X post succeeded
- article-writer (23:30) — Article generation succeeded
- larry-post-morning-en (07:30) — TikTok post succeeded
```

**Questions:**
- Why selective failures when all target the same #metrics channel?
- What distinguishes jobs with 6 consecutive errors vs 1 error?

## Step 1: Verify Execution vs Reporting Separation

First, confirm whether the task itself succeeded:

```bash
# Check if Larry TikTok posts were actually executed
ls -la ~/.openclaw/workspace/hooks/tiktok-slots/2026-03-16/

# Check if app-metrics data was generated
ls -la ~/.openclaw/workspace/app-metrics/2026-03-16/
```

**Result interpretation:**
- Files exist → Task execution succeeded, reporting layer failed
- Files missing → Task itself failed

In this case, Larry post files and app-metrics data existed. **Execution succeeded; only reporting failed.**

## Step 2: Analyze Selective Failure Patterns

Compare successful vs failed crons:

| cron | Result | Characteristics |
|------|--------|----------------|
| build-in-public | ✅ | Lightweight (1 X post, short message) |
| article-writer | ✅ | Lightweight (article generation, concise Slack report) |
| app-metrics | ❌ | Heavy (RevenueCat + ASC API calls, long report) |
| larry-post-* | ❌ | Heavy (image generation + Postiz API + detailed logs) |
| autonomy-check | ❌ | Heavy (multiple API checks + long report) |

**Pattern discovered:**
- Lightweight tasks (short reports, minimal external API calls) → Succeed
- Heavy tasks (long reports, 3+ API calls) → Fail

Hypothesis: **Slack messaging layer hits timeout or payload size limits.**

## Step 3: Check OpenClaw Message Tool Logs

```bash
# Check OpenClaw Gateway logs (Mac Mini example)
tail -100 ~/Library/Logs/openclaw/gateway.log | grep "message.*failed"

# Check specific cron execution logs
tail -50 ~/.openclaw/workspace/logs/app-metrics-morning-2026-03-16.log
```

**Expected information:**
- Timeout errors (`ETIMEDOUT`)
- Slack API error codes (`429 Too Many Requests`, `413 Payload Too Large`)
- OpenClaw message tool internal errors

## Step 4: Temporary Workarounds

If the root cause is in the Slack delivery layer:

### Workaround A: Shorten Reports

```javascript
// ❌ Long report (fails often)
await message.send({
  channel: 'slack',
  target: 'C091G3PKHL2',
  message: `📊 app-metrics completed\n\nDetailed report:\n${longReport}\n\nMRR: $${mrr}\nTrials: ${trials}\n...(1000+ lines)`
});

// ✅ Concise report (succeeds often)
await message.send({
  channel: 'slack',
  target: 'C091G3PKHL2',
  message: `📊 app-metrics done | MRR: $${mrr} | Trials: ${trials}\nDetails: workspace/app-metrics/${today}/`
});
```

### Workaround B: Skip Reporting Layer

Use `delivery.mode: "none"` in OpenClaw cron config:

```javascript
{
  "schedule": { "kind": "cron", "expr": "5 5 * * *" },
  "payload": { "kind": "agentTurn", "message": "Execute app-metrics skill" },
  "delivery": { "mode": "none" },  // Skip Slack reporting
  "sessionTarget": "isolated"
}
```

Note: This requires separate monitoring to track success/failure.

## Step 5: Root Cause Investigation (Long-term Fix)

To fully resolve this issue:

1. **Review OpenClaw message tool source code**
   - Read `~/.openclaw/node_modules/openclaw/src/tools/message.js`
   - Check Slack API call timeout settings

2. **Check Slack API limits**
   - Read [Slack API rate limits](https://api.slack.com/docs/rate-limits)
   - Check `chat.postMessage` limits (1 message/sec)

3. **Search OpenClaw Issues**
   - GitHub: [github.com/openclaw/openclaw/issues](https://github.com/openclaw/openclaw/issues)
   - Search: `"Message failed" delivery slack`

## Key Takeaways

| Lesson | Detail |
|--------|--------|
| Reporting failure ≠ Task failure | Always verify task execution by checking workspace/ files first |
| Selective failure patterns are clues | Compare success/failure to find common factors (size/API count/timeout) |
| Keep reports concise | Store detailed reports in files, send summaries to Slack |
| Track root causes systematically | Gateway logs → OpenClaw message tool → Slack API |

**Next steps:**
- Confirm OpenClaw message tool log format
- Manually re-run failed crons to test reproducibility
- Check if Slack API rate limits are being hit via `openclaw gateway` metrics

When encountering this issue, start by verifying whether execution succeeded. Reporting layer failures are easier to fix than task execution failures.
