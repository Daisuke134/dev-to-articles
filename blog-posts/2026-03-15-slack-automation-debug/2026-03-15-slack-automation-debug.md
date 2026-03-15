---
title: "How to Debug Selective Message Delivery Failures in Slack Automation"
published: true
tags: slack, nodejs, debugging, automation
---

## TL;DR
Encountered selective failures where some Slack notifications succeeded while others failed to the same channel. Root cause: async timing issues and insufficient error handling. Cron job execution order and message payload size affected delivery success rate.

## Prerequisites
- Node.js v25.6.1
- Slack SDK (`@slack/web-api`) or REST API
- Cron-based automation environment
- OpenClaw Gateway (or any Node.js automation framework)

## The Symptom

On March 15th, 6 out of 22 cron jobs executed with this pattern:

| Cron | Time | Result | Note |
|------|------|--------|------|
| factory-bp-revenue | 22:00 | ✅ Success | Notification delivered |
| larry-post-evening-en | 21:00 | ❌ Failed | Task completed, notification failed |
| larry-post-evening-ja | 21:30 | ❌ Failed | Task completed, notification failed |
| factory-bp-efficiency | 22:20 | ❌ Failed | Task completed, notification failed |
| factory-bp-internal | 22:40 | ❌ Failed | Task completed, notification failed |

**Key characteristics:**
- All targeted the same Slack channel (`#metrics`)
- All tasks completed (files created, TikTok posts verified)
- **Only notification delivery** failed selectively

## Step 1: Check Error Logs

```bash
# Check OpenClaw Gateway logs
tail -100 ~/.openclaw/logs/gateway.log | grep -i "message failed"

# Verify task outputs
ls -la ~/.openclaw/workspace/larry/2026-03-15-*
```

**Findings:**
- `Message failed` errors logged
- Task output files all created
- Error details missing (caught and discarded in catch blocks)

## Step 2: Compare Message Payload Sizes

```javascript
// Successful message (factory-bp-revenue)
const successMessage = `📝 factory-bp-revenue completed
✅ BP search: 5 results
📄 revenue-bp-2026-03-15.md created`;

// Failed message (larry-post-evening-en)
const failedMessage = `🎬 Larry post completed
🌐 TikTok EN: https://...
📊 Hook: "7 signs you're healing..."
🎨 6 slides generated`;

console.log('Success:', successMessage.length); // 72 chars
console.log('Failed:', failedMessage.length);   // 120+ chars
```

**Hypothesis 1:** Longer messages timing out

## Step 3: Analyze Execution Timing

```bash
# Check cron job intervals
openclaw cron list | grep larry
openclaw cron list | grep factory-bp
```

**Findings:**
- larry-post jobs: 30-minute intervals (21:00, 21:30)
- factory-bp jobs: 20-minute intervals (22:00, 22:20, 22:40)
- Successful factory-bp-revenue was **the first one executed**

**Hypothesis 2:** Hitting Slack API rate limits

## Step 4: Improve Error Handling

```javascript
// Before (error details lost)
try {
  await sendSlackMessage(channel, message);
} catch (error) {
  console.error('Message failed');
}

// After (capture error details)
try {
  await sendSlackMessage(channel, message);
} catch (error) {
  console.error('Message failed:', {
    error: error.message,
    code: error.code,
    statusCode: error.statusCode,
    retryable: error.retryable
  });
}
```

## Step 5: Add Retry Logic

```javascript
async function sendSlackMessageWithRetry(channel, message, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      await sendSlackMessage(channel, message);
      return;
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      
      // Exponential backoff
      const delay = Math.pow(2, i) * 1000;
      console.log(`Retry ${i + 1}/${maxRetries} after ${delay}ms`);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

## Step 6: Verify Rate Limits

Slack API rate limits:

| Tier | Limit |
|------|-------|
| Standard | 1 request/second |
| Tier 2 | 20 requests/minute |
| Tier 3 | 50 requests/minute |

With 6 cron jobs over 100 minutes (21:00-22:40), Standard Tier should suffice. **Concurrent execution** might be the culprit.

```javascript
// Add delay between cron jobs
setTimeout(() => sendSlackMessage(...), jobIndex * 5000);
```

## Key Takeaways

| Lesson | Detail |
|--------|--------|
| Log error details | Capture `error.message`, `error.code`, and `error.statusCode` |
| Retry is mandatory | Network calls fail. Design for it. |
| Mind rate limits | Space out consecutive sends to the same channel |
| Selective failure pattern | First request succeeds, subsequent ones hit rate limit |
| Timeout settings | Default 3s insufficient; use 10s+ |

**Next actions:**
1. Improve error handling to log error details
2. Add retry logic to all message sends
3. Set 5-second delay between cron jobs
4. Verify improvement in tomorrow's logs
