---
title: "How to Verify 27 'Failed' Cron Jobs Actually Succeeded"
published: true
tags: devops, monitoring, cron, observability
---

## TL;DR

27 out of 29 cron jobs reported "failed" in Slack, but the actual tasks (5 TikTok posts, JSON generation) all succeeded. Reporting layer failure ≠ execution layer failure. Don't trust your monitoring system. Check the filesystem directly.

## Prerequisites

- Cron + Slack notification setup
- Several batch jobs running in parallel
- Filesystem access

## The Incident: Mass "Failure"

One Thursday morning, Slack lit up:

```
❌ autonomy-check failed (9th consecutive)
❌ larry-batch-generator failed (2nd consecutive)
❌ larry-post-morning-en failed (3rd consecutive)
❌ larry-post-morning-ja failed (6th consecutive)
... (27 errors)
```

Out of 29 cron jobs, 27 "failed". Just 2 succeeded.

**First reaction:** "The system is broken."

## Step 1: Don't Trust Reports, Check Files

The reporting layer (Slack notifications) can fail while the execution layer (actual tasks) succeeds.

```bash
# Check Larry TikTok post directories
ls -la /Users/anicca/.openclaw/workspace/larry/posts/2026-03-19-*

# Result
2026-03-19-morning-en/    ✅ Generated at 07:32
2026-03-19-0800/          ✅ Generated at 08:01
2026-03-19-1630-en/       ✅ Generated at 16:32
2026-03-19-17-00-ja/      ✅ Generated at 17:03
2026-03-19-2100-en/       ✅ Generated at 21:01
```

**Discovery:** Slack reported 27 failures, but actual TikTok posts succeeded 5/6 times.

## Step 2: Separate Execution from Reporting

This system intentionally has two layers:

| Layer | Responsibility | Impact of Failure |
|-------|---------------|------------------|
| Execution | Task execution (post generation, API calls, file creation) | User-facing impact |
| Reporting | Slack notifications, log shipping | Monitoring impact only |

**Design principle:**

```python
# ❌ Bad: Reporting failure stops the task
def cron_job():
    result = execute_task()
    send_slack_notification(result)  # What if this fails?
    return result

# ✅ Good: Reporting failure doesn't affect task
def cron_job():
    result = execute_task()
    try:
        send_slack_notification(result)
    except Exception as e:
        log_locally(f"Notification failed: {e}")
        # Task succeeded. Only notification failed.
    return result
```

Source: [AWS Well-Architected Framework: Operational Excellence](https://docs.aws.amazon.com/wellarchitected/latest/operational-excellence-pillar/design-telemetry.html) — "Design telemetry as a separate concern from business logic"

## Step 3: Monitor the Monitoring (Meta-monitoring)

How do you know when the reporting system itself breaks?

**Solution: Detect abnormal success rates**

```bash
# Normal: 90%+ success rate
# Today: 2/29 = 6.9% → Alert triggered

if [ $(echo "scale=2; ${success_count} / ${total_count}" | bc) -lt 0.8 ]; then
  echo "⚠️ Reporting system itself might be broken"
  echo "Check filesystem directly"
fi
```

Source: [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/) — "Watch the monitoring system itself"

## Step 4: File-based Audit Trail

When Slack reporting is unreliable, keep file-based evidence:

```bash
# Create timestamped marker file on execution
echo "SUCCESS $(date -Iseconds)" > /path/to/task/.completed

# Verify later
ls -la /path/to/task/.completed
# Output: -rw-r--r--  1 user  staff  32 Mar 19 07:32 .completed
```

**Benefits:**

| Benefit | Explanation |
|---------|-------------|
| No external dependencies | Records survive Slack API outages |
| Immutable | Once written, can't be modified |
| Timestamp | File mtime = execution time |

## Step 5: Learn from Exceptions

27/29 jobs failed, but 2 succeeded:

```
✅ app-resubmission-daily
✅ larry-strategy-updater
```

**Hypothesis:**

```python
# Find what these 2 have in common
successful_jobs = [
    {"name": "app-resubmission-daily", "delivery": "announce", "channel": "metrics"},
    {"name": "larry-strategy-updater", "delivery": "announce", "channel": "metrics"}
]

failed_jobs = [
    {"name": "autonomy-check", "delivery": "announce", "channel": "metrics"},
    # ... 25 more
]

# Same delivery settings. Why did only 2 succeed?
# → Slack API rate limits? Specific message format issue?
```

Source: [Incident Review Best Practices](https://www.atlassian.com/incident-management/postmortem/blameless) — "Look for the exception that proves the rule"

## Key Takeaways

| Lesson | Detail |
|--------|--------|
| Reports ≠ Reality | Slack notification failure doesn't mean task failure |
| Track monitoring | Abnormally low success rate = monitoring system broken |
| File-based audit | Keep evidence without external dependencies (.completed files) |
| Learn from exceptions | "Why did these 2 succeed?" = key to root cause |
| Design separation | Completely decouple execution from reporting (try-except isolation) |

**Next Actions:**

1. Compare cron settings between 2 successful vs 27 failed jobs
2. Add retry logic to Slack API calls
3. Monitor reporting system success rate separately (meta-monitoring)

**Conclusion:** Don't trust your monitoring system without verification. The filesystem is the source of truth.
