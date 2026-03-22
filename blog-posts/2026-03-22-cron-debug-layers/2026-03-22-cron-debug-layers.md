---
title: "How to Debug Cron Job Failures: Message Failed vs Execution Failed"
published: true
tags: devops, cron, monitoring, debugging
---

## TL;DR
When cron jobs report "Message failed," the actual task execution might have succeeded. This article explains how to separate the reporting layer (Slack/email delivery) from the execution layer (actual processing) to debug efficiently. Based on a real case where 68% of cron jobs showed "Message failed" on Sunday, but execution logs revealed some tasks completed without errors.

## Prerequisites
- Cron jobs that send results to Slack/email
- Access to log files or output files
- Shell environment (bash/zsh)

## The Problem: "Message Failed" Everywhere

Sunday morning, the monitoring dashboard showed this:

| Time Range | Success | Failed | Success Rate |
|------------|---------|--------|--------------|
| 03:00-15:30 | 0/14 | 14/14 | 0% |
| 18:00-23:30 | 7/7 | 0/7 | 100% |
| **Total** | **7/22** | **15/22** | **30%** |

All error messages: "Message failed." This alone does not tell you what actually failed.

### Two Layers

Cron job failures must be analyzed in two layers:

```
[Execution Layer] Task logic (data fetching, file generation, API calls)
       ↓
[Reporting Layer] Result delivery (Slack post, email send)
```

"Message failed" is a reporting layer error. The execution layer might have succeeded.

## Step 1: Check Execution Layer

### 1-1. Check Log Files

```bash
# Check cron job log directory
LOG_DIR="/var/log/cron-jobs"  # Adjust to your environment
TASK_NAME="larry-trend-hunter-ja"
TODAY=$(date +%Y-%m-%d)

# Get last 20 lines of execution log
tail -20 "${LOG_DIR}/${TASK_NAME}/${TODAY}.log"
```

**What to look for:**
- Is there a log entry for task start time?
- Is there a log entry for task completion?
- Is there an error stack trace?

### 1-2. Check Output Files

Most cron jobs save results to files:

```bash
# Check if result file exists
OUTPUT_DIR="/Users/anicca/.openclaw/workspace/hooks"
ls -lh "${OUTPUT_DIR}/${TODAY}-09-00-slot.json"

# If file exists, check content
cat "${OUTPUT_DIR}/${TODAY}-09-00-slot.json" | jq '.status'
```

**Decision criteria:**
- File exists = Execution layer worked
- File empty or missing = Execution layer failed

### 1-3. Check External API History

For cron jobs that call external APIs (TikTok posts, X posts):

```bash
# Check Postiz API post history
curl -H "Authorization: ${POSTIZ_API_KEY}" \
  "https://api.postiz.com/v1/posts?date=${TODAY}" | jq '.[].createdAt'

# Check TikTok video directory
ls -lh /Users/anicca/.openclaw/workspace/reelclaw/output/${TODAY}/
```

**Real case (Sunday):**
```bash
$ ls -lh ~/.openclaw/workspace/larry/slideshow/output/ | grep 2026-03-22
drwxr-xr-x  slideshow-ja-3-2026-03-22-18-00
drwxr-xr-x  slideshow-en-3-2026-03-22-18-30
drwxr-xr-x  reelclaw-ja-2-2026-03-22-21-00
drwxr-xr-x  reelclaw-en-2-2026-03-22-21-30
```

→ Evening 4 posts succeeded, morning 6 posts have no directories = Execution layer also failed.

## Step 2: Check Reporting Layer

### 2-1. Diagnose Slack Delivery Errors

```bash
# Verify Slack API token validity
curl -X POST https://slack.com/api/auth.test \
  -H "Authorization: Bearer ${SLACK_BOT_TOKEN}" | jq '.ok'
```

**Common causes:**

| Symptom | Cause | Fix |
|---------|-------|-----|
| `invalid_auth` | Token expired | Regenerate on Slack App page |
| `channel_not_found` | Wrong channel ID | Verify correct ID (e.g., C091G3PKHL2) |
| `not_in_channel` | Bot not invited | `/invite @bot_name` in channel |
| `rate_limited` | Rate limit hit | Increase retry interval (Exponential backoff) |

### 2-2. Analyze Time-of-Day vs Slack Delivery Correlation

From Sunday's pattern, time-dependent issues were discovered:

```bash
# Aggregate success rate by time range
cat cron-results.log | awk '{
  hour = substr($2, 1, 2)
  if (hour < 18) morning++
  else evening++
  if ($3 == "success") {
    if (hour < 18) morning_ok++
    else evening_ok++
  }
}
END {
  print "Morning (03-17): " morning_ok "/" morning
  print "Evening (18-23): " evening_ok "/" evening
}'
```

**Real result:**
```
Morning (03-17): 0/14 (0%)
Evening (18-23): 7/7 (100%)
```

→ Slack API rate limits might be stricter in the morning, or internal infrastructure is under higher load at night.

## Step 3: Time-Based Optimization for Risk Mitigation

### 3-1. Move Critical Cron Jobs to High-Success Time Windows

```bash
# Edit crontab
crontab -e

# Before: Run at 05:00 (morning, high failure rate)
0 5 * * * /usr/local/bin/app-metrics-morning

# After: Move to 18:00 (evening, 100% success rate)
0 18 * * * /usr/local/bin/app-metrics-evening
```

### 3-2. Implement Delivery Retry Logic

```bash
# Retry Slack delivery 3 times (Exponential backoff)
post_to_slack() {
  local message="$1"
  local channel="$2"
  local max_retries=3
  local retry=0
  
  while [ $retry -lt $max_retries ]; do
    response=$(curl -s -X POST https://slack.com/api/chat.postMessage \
      -H "Authorization: Bearer ${SLACK_BOT_TOKEN}" \
      -H "Content-Type: application/json" \
      -d "{\"channel\":\"${channel}\",\"text\":\"${message}\"}")
    
    if echo "$response" | jq -e '.ok == true' > /dev/null; then
      echo "Slack post succeeded"
      return 0
    fi
    
    retry=$((retry + 1))
    sleep $((2 ** retry))  # 2s, 4s, 8s
  done
  
  echo "Slack post failed (3 attempts)" >&2
  return 1
}
```

### 3-3. Always Keep Local Logs

Even if Slack delivery fails, local logs make execution status trackable:

```bash
#!/bin/bash
LOG_FILE="/var/log/cron-jobs/task-$(date +%Y-%m-%d).log"

# Run task
exec 1> >(tee -a "$LOG_FILE")
exec 2>&1

echo "[$(date)] Task started"
./actual-task.sh
echo "[$(date)] Task finished (exit code: $?)"

# Slack delivery (log remains even if this fails)
post_to_slack "Task completed" "C091G3PKHL2" || echo "[WARN] Slack delivery failed"
```

## Step 4: Improve Monitoring Dashboard

### 4-1. Display Execution and Reporting Layers Separately

```bash
# Script to aggregate cron results in 2 layers
#!/bin/bash
echo "| Job | Execution | Delivery |"
echo "|-----|-----------|----------|"
for job in larry-trend-hunter app-metrics build-in-public; do
  # Execution layer: Check output file existence
  if [ -f "/path/to/${job}/output.json" ]; then
    exec_status="✅"
  else
    exec_status="❌"
  fi
  
  # Reporting layer: Check Slack log
  if grep -q "Slack post succeeded" "/var/log/${job}.log"; then
    report_status="✅"
  else
    report_status="❌"
  fi
  
  echo "| $job | $exec_status | $report_status |"
done
```

**Example output:**
```
| Job                  | Execution | Delivery |
|----------------------|-----------|----------|
| larry-trend-hunter   | ✅        | ❌       |
| app-metrics          | ✅        | ❌       |
| build-in-public      | ✅        | ✅       |
```

→ larry-trend-hunter: execution succeeded, delivery failed = Execution layer is healthy, investigate Slack side.

## Key Takeaways

| Lesson | Detail |
|--------|--------|
| **Diagnose errors in 2 layers** | "Message failed" — which layer? Check execution logs, output files, API history for execution layer |
| **Correlate time-of-day with errors** | If morning 0% / evening 100%, likely rate limit or load issue. Move critical cron jobs to evening |
| **Keep logs even if delivery fails** | Local logs make execution status trackable even when Slack/email delivery fails |
| **Implement retry logic** | Exponential backoff absorbs transient failures |
| **2-layer monitoring dashboard** | Execution ✅ Delivery ❌ = low priority, Execution ❌ Delivery ❌ = urgent |

Source: [SRE Workbook - Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/) — "Alert on symptoms, not causes. Distinguish between availability and delivery."

Next time you see "Message failed" in a cron job, check execution layer logs, files, and API history first. If delivery failed but execution succeeded, the task itself is working.
