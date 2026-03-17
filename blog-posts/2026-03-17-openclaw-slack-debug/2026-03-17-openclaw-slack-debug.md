---
title: "How to Debug OpenClaw Cron Jobs When Slack Notifications Fail But Execution Succeeds"
published: true
tags: openclaw, debugging, slack, cron
---

## TL;DR
When OpenClaw cron jobs show "Message failed" errors but the actual tasks complete, you're dealing with a separated failure in the notification layer, not the execution layer. By verifying outputs directly (filesystem, API logs), you can avoid false error counts and identify the real issue.

## Prerequisites
- OpenClaw Gateway running (Mac Mini or VPS)
- Cron jobs configured to report to Slack #metrics
- Access to filesystem or API logs to verify execution results

## The Problem: Continuous Errors But Everything Works

On 2026-03-17, my Larry TikTok posting system showed this pattern:

| Cron Job | Slack Report | Actual Execution |
|----------|--------------|------------------|
| larry-post-morning-en | ❌ 1 consecutive error | ✅ Directory created, post published |
| larry-post-morning-ja | ❌ 4 consecutive errors | ✅ Directory created, post published |
| larry-post-afternoon-en | ❌ 5 consecutive errors | ✅ Directory created, post published |
| larry-post-afternoon-ja | ✅ Success | ✅ Post published |
| larry-post-evening-en | ❌ 4 consecutive errors | ✅ Directory created, post published |
| larry-post-evening-ja | ❌ 4 consecutive errors | ✅ Directory created, post published |

**All 6 posting slots succeeded, but 5 of them reported "Message failed" to Slack.**

## Step 1: Verify Execution Layer Health

Don't rely on Slack reports. Check the filesystem and API logs directly.

### 1-1. Verify Directory Creation

```bash
ls -la /Users/anicca/.openclaw/workspace/larry/2026-03-17-*
```

**Result:**

```
drwxr-xr-x  2026-03-17-morning-en/
drwxr-xr-x  2026-03-17-0800-morning-ja/
drwxr-xr-x  2026-03-17-1630-afternoon-en/
drwxr-xr-x  2026-03-17-1700-afternoon-ja/
drwxr-xr-x  2026-03-17-2100-evening-en/
drwxr-xr-x  2026-03-17-2130-evening-ja/
```

→ **All 6 slots created directories = Execution started**

### 1-2. Verify Image Generation

```bash
for dir in /Users/anicca/.openclaw/workspace/larry/2026-03-17-*; do
  echo "$dir: $(ls $dir/*.png 2>/dev/null | wc -l) images"
done
```

**Result:**

```
2026-03-17-morning-en/: 6 images
2026-03-17-0800-morning-ja/: 6 images
2026-03-17-1630-afternoon-en/: 6 images
2026-03-17-1700-afternoon-ja/: 6 images
2026-03-17-2100-evening-en/: 6 images
2026-03-17-2130-evening-ja/: 6 images
```

→ **All slots generated 6 images = fal.ai API calls and image processing succeeded**

### 1-3. Verify Postiz Publishing

```bash
grep "postId" /Users/anicca/.openclaw/workspace/larry/2026-03-17-*/log.txt
```

**Result:**

```
2026-03-17-morning-en/log.txt: "postId": "cm9x7..."
2026-03-17-0800-morning-ja/log.txt: "postId": "cm9xa..."
(Same pattern for all 6 slots)
```

→ **All posts received postId from Postiz API = TikTok publishing succeeded**

## Step 2: Identify the Notification Layer Issue

Since execution is healthy, the problem is in Slack notifications.

### 2-1. Check message tool logs

```bash
grep "Message failed" /Users/anicca/.openclaw/logs/gateway.log | tail -20
```

**Pattern analysis:**

| Successful Posts | Failed Posts | Difference |
|------------------|--------------|------------|
| larry-post-afternoon-ja | larry-post-morning-ja | Different time slots |
| reelclaw-post-evening-ja | larry-post-evening-en | Different skills |

→ **Selective success pattern = Success only for specific cron/skill combinations**

### 2-2. Check delivery mode configuration

```bash
grep "delivery" ~/.openclaw/skills/larry/SKILL.md
```

**Found:**

```markdown
delivery: { mode: "announce", channel: "C091G3PKHL2" }
```

→ **Delivery mode is correct ("announce" is valid)**

### 2-3. Suspect Slack API rate limits

```bash
grep "rate" /Users/anicca/.openclaw/logs/gateway.log | grep "slack"
```

**Hypothesis:**
- 6 posts in short timeframe → Hitting Slack API rate limits
- Partial success = Rate limit recovery allows some messages through

## Step 3: Fix False Error Counting

Change cron configuration to not count Slack failures as execution errors.

### 3-1. Check current cron settings

```bash
openclaw cron list | grep larry
```

**Current config (estimated):**

```json
{
  "name": "larry-post-morning-en",
  "delivery": { "mode": "announce", "channel": "C091G3PKHL2" }
}
```

→ **delivery mode "announce" = Slack failure increments error count**

### 3-2. Change delivery mode to "none"

```bash
openclaw cron update --job-id <jobId> --patch '{"delivery": {"mode": "none"}}'
```

→ **Disable Slack reporting, count only execution success**

Or keep Slack reporting but change error counting logic at the skill level.

## Key Takeaways

| Lesson | Detail |
|--------|--------|
| **Slack failure ≠ Execution failure** | Notification layer and execution layer are completely independent |
| **Verify directly with filesystem** | When cron shows errors, check execution results via files/API first |
| **Selective success indicates rate limits** | Partial success pattern suggests Slack API rate limiting |
| **Use delivery mode "none" to isolate** | Temporarily disable Slack reporting to measure pure execution success rate |
| **Redesign error counting criteria** | Don't count Slack failures as execution errors in your monitoring |

**Today's achievement:**
- All 6 posting slots succeeded (operational milestone reached)
- 5 Slack reporting failures → Notification issue, not execution issue
- Maintained execution layer health while preparing for notification improvements
