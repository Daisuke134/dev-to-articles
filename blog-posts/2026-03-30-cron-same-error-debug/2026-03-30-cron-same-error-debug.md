---
title: "How to Debug Silent Cron Failures When All Errors Share the Same Root Cause"
published: true
tags: cron, devops, monitoring, debugging
---

## TL;DR

9 out of 26 cron jobs failed with the identical error message: "Message failed." Instead of investigating each job individually, comparing failed jobs against successful ones revealed the root cause in minutes: the Slack delivery layer was broken, not the jobs themselves.

## Prerequisites

- A system running multiple cron jobs (10+)
- Job execution and result notification handled by separate layers
- Notification channel: Slack (or any external messaging service)

## Step 1: Check if Errors Share a Pattern

List every failed job and its error message side by side.

```
autonomy-check       → ⚠️ Message failed
trend-hunter-ja      → ⚠️ Message failed
app-metrics-morning  → ⚠️ Message failed
latest-papers        → ⚠️ Message failed
suffering-detector   → ⚠️ Message failed
... (9 total)
```

All 9 failures produce the exact same error string. This immediately shifts the hypothesis from "individual job bugs" to "shared infrastructure problem."

## Step 2: Diff Successful Jobs Against Failed Ones

| Attribute | Successful (17) | Failed (9) |
|-----------|-----------------|------------|
| Execution duration | Normal range | Normal range |
| Output destination | Direct API calls (Postiz, git push) | Slack delivery only |
| Job category | Content generation | Analytics, trends, audits |

The pattern is clear: **successful jobs post directly to external APIs. Failed jobs depend entirely on Slack delivery.**

## Step 3: Isolate the Failing Layer

Break down the cron execution flow:

```
[cron scheduler] → [job execution] → [result delivery (Slack)]
```

Failed jobs had normal execution durations, meaning `[job execution]` completed successfully. The failure happened at `[result delivery]`.

This is a critical insight: the jobs actually ran. Their output just never reached Slack.

## Step 4: Narrow Down the Root Cause

With the problem isolated to the Slack delivery layer, check:

1. **Bot token expiry** — Has the Slack token been revoked or expired?
2. **Channel permissions** — Is the bot still a member of the target channel?
3. **Rate limiting** — Are too many messages sent in a short window?
4. **Delivery configuration** — Is the cron delivery mode set correctly?

In this case, the delivery configuration needed a review.

## Step 5: Prioritize by Consecutive Failure Count

| Job | Consecutive failures | Priority |
|-----|---------------------|----------|
| trend-hunter-ja | 13 days | Critical |
| factory-bp-internal | 6 days | High |
| app-metrics-morning | 6 days | High |
| factory-bp-revenue | 5 days | Medium |
| autonomy-check | 4 days | Medium |

A job failing for 13 consecutive days will not self-heal. Track consecutive failure counts to distinguish transient issues from structural problems.

## Key Takeaways

| Lesson | Detail |
|--------|--------|
| Same error everywhere = shared layer problem | Investigating individual jobs wastes time when the root cause is upstream |
| Diff success vs failure | Comparing what works against what fails reveals the broken layer faster than any log dive |
| Separate execution from delivery | Normal duration + delivery failure = the job is innocent |
| Track consecutive failure days | Problems that persist for days are structural, not transient |
