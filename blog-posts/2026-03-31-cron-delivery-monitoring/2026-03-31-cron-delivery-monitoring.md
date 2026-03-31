---
title: "How to Separate Delivery Failures from Execution Failures in Cron Jobs"
published: true
tags: cron, monitoring, devops, observability
---

## TL;DR

When your cron job reports "failed," it might have executed perfectly — only the notification delivery (Slack, email, webhook) failed. Monitoring execution and delivery as separate layers prevents false alarms and unnecessary re-runs.

## The Problem: 9 "Broken" Jobs That Were Fine

I run 43 cron jobs on OpenClaw for content generation, analytics, and infrastructure tasks. One day, 9 jobs reported "Message failed" errors. Meanwhile, all 14 content generation jobs succeeded.

```
build-in-public    | Message failed | 1 consecutive
larry-trend-hunter | Message failed | 14 consecutive  ← 3 weeks!
app-metrics        | Message failed | 7 consecutive
```

My first instinct: "9 jobs are broken, fix them." But when I investigated, every job had completed its actual work. Only the Slack notification step was failing.

## Why This Happens

A typical cron job flow:

```
[Execute job] → [Generate output] → [Send notification] → [Report status]
```

"Message failed" occurs at step 3. But most cron schedulers report the final step's result as the job's status. So a delivery failure becomes a "job failure."

This conflation causes two problems:
1. **False alarms**: You investigate jobs that are working fine
2. **Missed fixes**: The real issue (delivery infrastructure) gets buried under "job failure" noise

## The Fix: Three-Layer Monitoring

### Layer 1: Execution (Did the job run?)

```bash
RESULT=$(run_job 2>&1)
EXIT_CODE=$?
echo "{\"job\":\"$JOB_NAME\",\"exit_code\":$EXIT_CODE,\"ts\":\"$(date -u +%FT%TZ)\"}" \
  >> /var/log/cron-execution.jsonl
```

### Layer 2: Artifact (Did it produce the expected output?)

```bash
EXPECTED="/workspace/output/${TODAY}/result.json"
if [ -f "$EXPECTED" ]; then
  ARTIFACT_STATUS="success"
else
  ARTIFACT_STATUS="missing"
fi
```

### Layer 3: Delivery (Did the notification reach its destination?)

```bash
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
  -X POST "$SLACK_WEBHOOK" \
  -H "Content-Type: application/json" \
  -d "{\"text\":\"$MSG\"}")

if [ "$HTTP_CODE" != "200" ]; then
  # Log delivery failure separately — do NOT mark the job as failed
  echo "{\"job\":\"$JOB_NAME\",\"delivery\":\"failed\",\"http\":$HTTP_CODE}" \
    >> /var/log/cron-delivery.jsonl
fi
```

### Alert Routing by Layer

| Layer | On Failure | Severity |
|-------|-----------|----------|
| Layer 1 (Execution) | Alert immediately, consider re-run | 🔴 High |
| Layer 2 (Artifact) | Alert, inspect logs | 🟡 Medium |
| Layer 3 (Delivery) | Retry delivery only. Do NOT re-run the job | 🟢 Low |

## Results After Separation

| Metric | Before | After |
|--------|--------|-------|
| "Failed" jobs per day | 9 | 0 (execution failures) |
| False alarms per day | 9 | 0 |
| Delivery issues detected | Unknown (mixed in) | 9 (isolated as Slack layer problem) |

The trend-hunter job with 14 consecutive "errors" had been executing correctly every single time. A structural issue in the Slack delivery layer had gone unnoticed for 3 weeks because it was classified as a "job failure."

## Key Takeaways

| Lesson | Detail |
|--------|--------|
| Decompose "failure" | Monitor execution, artifact, and delivery as independent layers |
| Delivery failure ≠ execution failure | Your job may have succeeded even when no notification arrives |
| Look at the pattern of consecutive errors | 14 consecutive identical errors point to infrastructure, not the job |
| Define re-run criteria | Only re-run on Layer 1 failure. Re-running for Layer 3 failure wastes resources |
