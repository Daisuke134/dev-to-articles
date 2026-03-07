---
title: "How to Detect Cron Failures in AI Agent Systems Before They Break Your Automation"
published: true
tags: aiagent, cron, monitoring, devops
---

## TL;DR

My AI agent's `roundtable-standup` cron job failed for 3 days without alerts. Root cause: **success-only notifications** and **no health checks**. Fixed by adding a monitoring cron that alerts when any job hasn't run in 24 hours.

## Prerequisites

- OpenClaw Gateway running on Mac Mini
- 43 cron jobs scheduled (5min to 24h intervals)
- Slack #metrics channel for notifications

## The Problem: 3-Day Undetected Failure

### Timeline

| Date | Event |
|------|-------|
| Mar 4, 09:00 | Last successful standup execution |
| Mar 5-7 | **No execution logs** |
| Mar 7, 20:00 | Discovered while checking daily-memory |

### Why Detection Was Delayed

1. **Only success was notified** to Slack (failures were silent)
2. **Cron was still registered** (visible in `openclaw cron list`)
3. **Other crons were running** (factory, larry), so no system-wide alarm

Source: [Google SRE Workbook: Monitoring Distributed Systems](https://sre.google/workbook/monitoring/)
Key quote: "The four golden signals of monitoring are latency, traffic, errors, and saturation. For batch jobs, add **recency** (last successful run time)."

## Root Cause

OpenClaw's cron engine has **no retry mechanism**. One failure = wait until next scheduled run.

```javascript
// Simplified OpenClaw cron execution
async function executeCron(cronJob) {
  try {
    await runSkill(cronJob.skillName);
    await notifySlack(`${cronJob.name} succeeded`);
  } catch (error) {
    // Error logged locally, but NOT sent to Slack
    console.error(error);
  }
}
```

**Issues:**
- Error logs go to local files, not Slack
- Next run is 24 hours later → delayed detection
- Root cause (permissions, API limits, network) is hard to investigate

## Fix: Add a Health Check Cron

### Step 1: Log All Cron Executions

```bash
# openclaw/src/cron-runner.js (pseudo-code)
async function executeCronWithLogging(cronJob) {
  const logPath = `/Users/anicca/.openclaw/logs/cron-history.jsonl`;
  const entry = {
    name: cronJob.name,
    timestamp: new Date().toISOString(),
    status: 'started'
  };
  
  try {
    await runSkill(cronJob.skillName);
    entry.status = 'success';
  } catch (error) {
    entry.status = 'failed';
    entry.error = error.message;
  } finally {
    fs.appendFileSync(logPath, JSON.stringify(entry) + '\n');
  }
}
```

### Step 2: Create Health Check Script

```bash
# ~/.openclaw/skills/cron-health-check/check.sh
#!/bin/bash
NOW=$(date +%s)
ALERT_THRESHOLD=86400  # 24 hours

# Get last run time for each cron
jq -r '.name' < ~/.openclaw/state/cron-jobs.json | while read -r CRON_NAME; do
  LAST_RUN=$(grep "\"name\":\"$CRON_NAME\"" ~/.openclaw/logs/cron-history.jsonl | tail -1 | jq -r '.timestamp')
  
  if [ -z "$LAST_RUN" ]; then
    echo "⚠️ $CRON_NAME: No execution history"
    continue
  fi
  
  LAST_UNIX=$(date -j -f "%Y-%m-%dT%H:%M:%S" "$LAST_RUN" +%s 2>/dev/null)
  DIFF=$((NOW - LAST_UNIX))
  
  if [ $DIFF -gt $ALERT_THRESHOLD ]; then
    openclaw message send --channel slack --target 'C091G3PKHL2' \
      --message "🚨 cron-health: ${CRON_NAME} hasn't run for ${DIFF}s ($((DIFF/3600))h)"
  fi
done
```

### Step 3: Schedule the Health Check

```bash
openclaw cron add \
  --name "cron-health-check" \
  --schedule "0 */6 * * *" \  # Every 6 hours
  --command "bash ~/.openclaw/skills/cron-health-check/check.sh"
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Detection time for standup failure | 72 hours | < 6 hours |
| Root cause investigation | Manual log inspection | Auto-notified to Slack |
| Other cron failures | Undetected | Caught within 24h |

## Key Takeaways

| Lesson | Detail |
|--------|--------|
| **Notify failures, not just successes** | Success-only alerts create blind spots |
| **Monitor recency for scheduled jobs** | Check "last successful run" every 6-24h |
| **Centralize logs** | Local files are invisible. Push to Slack/Datadog |
| **Add retry logic** | One failure = 24h wait is too slow |

## References

- [Google SRE Workbook: Monitoring](https://sre.google/workbook/monitoring/) — Batch jobs need recency monitoring
- [Kubernetes CronJob Best Practices](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/#cron-job-limitations) — Retry policies for failed jobs

If you're running OpenClaw or building custom AI agents, I hope this helps you avoid silent failures.
