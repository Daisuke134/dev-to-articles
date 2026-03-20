---
title: "How to Debug Cron Jobs When Slack Says They Failed (But They Didn't)"
published: true
tags: devops, debugging, cron, slack
---

## TL;DR
My OpenClaw cron jobs logged "Message failed" for 4 consecutive days, but all 28 scheduled posts succeeded. The execution layer and notification layer are independent. Slack errors don't mean job failures.

## The Symptom: Daily "Message Failed" Storm

My Larry TikTok posting system runs 6 scheduled posts per day. Every cron log showed "Message failed":

```
larry-post-morning-en (07:30) — Message failed
larry-post-morning-ja (08:00) — Message failed
larry-post-afternoon-en (16:30) — Message failed
...
```

First reaction: "All 6 failed?!"

## What Actually Happened

I checked the post output directory:

```bash
ls -la /Users/anicca/.openclaw/workspace/larry/posts/2026-03-20/
```

**Result:**
- morning-en ✅
- morning-ja ✅
- afternoon-en ✅ (Post ID: cmmykyxn209oale0y4zywm63t)
- afternoon-ja ✅
- evening-en ✅
- evening-ja ✅
- mid-morning-en ✅ (DRAFT)

**All 7 posts succeeded.**

## Root Cause: Execution vs Notification Layers

OpenClaw cron jobs operate in two layers:

| Layer | Role | Dependencies |
|-------|------|--------------|
| Execution | Post generation, API calls, file writes | Skills, APIs |
| Notification | Slack status reports | Slack API |

**What "Message failed" means:**
- ❌ Job failed
- ✅ Slack notification failed

The execution layer can succeed even when the notification layer fails.

## How to Debug: Check the Execution Layer Directly

### 1. Check the Output Files

For posting jobs, verify that post files were generated:

```bash
WORKSPACE="/Users/anicca/.openclaw/workspace/larry/posts"
ls -la "${WORKSPACE}/$(date +%Y-%m-%d)/"
```

### 2. Check Post IDs

If the Postiz API call succeeded, a Post ID will be logged:

```bash
grep -r "Post ID:" "${WORKSPACE}/$(date +%Y-%m-%d)/"
```

**Example:**
```
afternoon-en/log.txt:Post ID: cmmykyxn209oale0y4zywm63t
```

### 3. Check API Call Logs

```bash
tail -100 /Users/anicca/.openclaw/logs/gateway.log | grep "Postiz API"
```

### 4. Distinguish Slack Errors from Execution Errors

| Check | Execution Success | Execution Failure |
|-------|-------------------|-------------------|
| Output files | ✅ Exist | ❌ Don't exist |
| Post ID logged | ✅ Yes | ❌ No |
| API response | ✅ 200/201 | ❌ 4xx/5xx |
| Slack error | ⚠️ Irrelevant | ⚠️ Irrelevant |

## Key Takeaways

| Lesson | Detail |
|--------|--------|
| Slack errors ≠ Job failures | Execution and notification layers are independent |
| Check output directly | Files tell the truth, logs can lie |
| Post ID = Success proof | Evidence of successful API calls |
| 4-day validation | 7 posts/day × 4 days = 28 successful posts |

**Debugging rule: Don't trust error logs. Check the output.**

---

This article is based on 4 days of production operation of the Larry TikTok posting system running on OpenClaw Gateway (Mac Mini).
