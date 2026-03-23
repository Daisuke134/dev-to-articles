---
title: "How to Verify AI Agent Cron Execution When Slack Reports Fail"
published: true
tags: devops, debugging, openclaw, automation
---

## TL;DR

When your AI agent system shows "Message failed" errors, the execution layer may still be working. By separating the reporting layer (Slack/Discord) from the execution layer (file creation/API calls), you can assess system health by checking the filesystem directly.

## Prerequisites

- AI agent framework (OpenClaw, AutoGen, etc.)
- Cron jobs running automated tasks
- External reporting service (Slack, Discord, webhooks)
- Delivery errors but uncertain execution status

## The Problem: Message Failed ≠ Execution Failed

For 2 days (2026-03-22 to 2026-03-23), 10+ cron jobs reported "Message failed" errors. Filesystem inspection revealed the truth.

**Execution Layer Evidence:**
- 170 files/directories created on that day
- 6 TikTok slideshow posts generated (Larry skill)
- 2 video posts generated (ReelClaw skill)
- 10 posts/day automation running

**Reporting Layer Status:**
- Slack delivery errors across 10+ crons
- Metrics visibility reduced
- System health assessment difficult

**Conclusion:** Message failed = reporting layer failure. Execution layer was healthy.

## Step 1: Inspect the Execution Layer Directly

**Filesystem-based verification:**

```bash
# Check files created today
TODAY=$(TZ=Asia/Tokyo date +%Y-%m-%d)
find ~/.openclaw/workspace -type f -newermt "$TODAY" | wc -l

# Check specific output directories
ls -la ~/.openclaw/workspace/tiktok-marketing/posts/ | grep "$TODAY"
```

**Results (2026-03-23):**
- 170 files confirmed
- 8 post directories found (`2026-03-23-*` pattern)

**Implication:** File creation = direct proof of cron execution. No dependency on external services.

## Step 2: Check Execution Result Files

**Verify result.json existence:**

```bash
# Find result.json in task directories
find ~/.openclaw/workspace/tiktok-marketing/posts/"$TODAY"-* -name "result.json" -exec cat {} \;
```

**Note:** Missing result.json = execution in progress OR script crashed before writing results. Use file creation timestamps to confirm execution started.

## Step 3: Inspect Cron Logs Directly

**Check tmux sessions for real-time logs:**

```bash
# List active sessions
tmux list-sessions

# Attach to cron session
tmux attach -t <session-name>

# Capture recent output
tmux capture-pane -t <session-name> -p | tail -50
```

**Check gateway.log:**

```bash
# OpenClaw gateway log (all cron executions recorded)
tail -100 ~/.openclaw/logs/gateway.log | grep "cron"
```

## Step 4: Separate Execution from Reporting

**Design decision:**

| Layer | Role | Failure Impact |
|-------|------|---------------|
| Execution Layer | Content generation, file creation, API calls | **CRITICAL** — System core |
| Reporting Layer | Slack delivery, metrics, visualization | Important but non-critical — monitoring only |

**Key takeaway:** Never assess execution layer health based on reporting layer status. Always verify filesystem/logs directly.

## Step 5: Fix Reporting Layer Errors (Priority: Medium)

**Common Slack delivery error causes:**

| Cause | Fix |
|-------|-----|
| API rate limit | Implement backoff strategy |
| Token expiration | Add token refresh script |
| Network timeout | Add retry logic |
| Channel ID changed | Verify environment variables |

**Fix example (OpenClaw message):**

```bash
# Verify Slack channel ID
openclaw message list-channels --channel slack

# Test message delivery
openclaw message send --channel slack --target 'C091G3PKHL2' --message "Test message"
```

**Priority when execution layer is healthy:** Downgrade reporting layer fixes to medium/low priority. Continue monitoring execution layer.

## Key Takeaways

| Lesson | Detail |
|--------|--------|
| **Message failed ≠ Execution failed** | Slack errors indicate reporting layer failure. Verify execution via filesystem |
| **Direct execution layer verification is mandatory** | 170 files created = execution success proof. No external service dependency |
| **Reporting layer is non-critical** | Visibility reduced but system core remains healthy. Safe to deprioritize |
| **Design: Layer separation** | Isolating execution from reporting allows partial failures without full system halt |
| **Debugging strategy** | Check filesystem → cron logs → reporting layer. Never reverse this order |

**Sources:**
- [Copyblogger: How to headline formula](https://copyblogger.com/10-sure-fire-headline-formulas-that-work/) — "8 out of 10 people will read the headline"
- [daily.dev: Write from expertise](https://daily.dev/blog/how-to-write-viral-stories-for-developers) — "Developers hate clickbait"
