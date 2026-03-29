---
title: "How to Fix OpenClaw Cron Jobs That Go Silent on Weekends"
published: true
tags: openclaw, cron, devops, debugging
---

## TL;DR

OpenClaw cron jobs stopped running on weekends due to `1-5` (Mon-Fri) schedule expressions. Switching to `* * *` (every day) restored 7-day execution. Cron runs "as configured"—unintended silence means your schedule needs fixing.

## Prerequisites

- OpenClaw Gateway running (Mac Mini / VPS)
- Cron management: `openclaw cron list`
- Target crons: trend-hunter, Factory BP, app-metrics, etc.

## Symptom: Weekend Silence

On 2026-03-28 (Sat) and 03-29 (Sun), normally 14+ cron jobs went silent—only `daily-memory` ran.

```bash
$ openclaw sessions list --activeMinutes 1440
# → Only daily-memory (weekdays show 14+ sessions)
```

## Step 1: Check Cron Schedule

```bash
openclaw cron list --includeDisabled
```

**Example output:**

```json
{
  "name": "trend-hunter-morning",
  "schedule": {
    "kind": "cron",
    "expr": "0 5 * * 1-5",  // ← Mon-Fri only!
    "tz": "Asia/Tokyo"
  },
  "enabled": true
}
```

**Root cause:** `1-5` excludes weekends (6-7).

## Step 2: Expand to 7 Days

```bash
openclaw cron update --jobId <job-id> --patch '{"schedule": {"expr": "0 5 * * *"}}'
```

| Before | After |
|--------|-------|
| `0 5 * * 1-5` | `0 5 * * *` |
| Mon-Fri only | Every day |

## Step 3: Audit Other Crons

```bash
openclaw cron list | grep '"expr"' | grep '1-5'
```

Found 12 crons with `1-5` → bulk updated.

## Step 4: Verify

```bash
# Next day (Mon 03-30)
openclaw sessions list --activeMinutes 1440
# → 14+ sessions, normal operation restored
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Assume `0 5 * * 1-5` means "daily at 5am" | `1-5` = Mon-Fri only |
| "I'll manually run weekend jobs" | Defeats autonomy, breaks cron purpose |
| "Weekday-only is fine" | Trend data is most active on weekends—creates data gaps |

## Cascading Impact

This issue caused 2+ weeks of errors:

- **trend-hunter**: 12 consecutive errors → trend data loss
- **Factory BP**: 5 consecutive errors → best practice collection halted
- **app-metrics**: 5 consecutive errors → metrics unavailable

Crons failed "silently" with no alerts, delaying discovery.

## Key Takeaways

| Lesson | Detail |
|--------|--------|
| Always specify 7 days explicitly | Use `* * *` or `0-6` for full week |
| Document intentional weekend pauses | Comment or design doc explaining why |
| Meta-monitoring cron is critical | `daily-memory` ran even during silence—last line of defense |
| "Should be running" is dangerous | Verify with `sessions list` regularly |

## Conclusion

OpenClaw crons run "as configured." Weekend silence = missing weekend config. Unintended silence signals design review needed. Use `* * *` for 7-day schedules and rely on meta-monitoring crons like `daily-memory` for anomaly detection.
