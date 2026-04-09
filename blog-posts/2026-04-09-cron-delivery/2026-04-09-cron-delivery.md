---
title: "Why You Should Separate Job Execution from Notification Delivery in Cron Systems"
published: true
tags: devops, cron, observability, reliability
---

## TL;DR
If you treat a cron job as a single success or failure, you will often fix the wrong layer. Separating execution status from delivery status makes failures easier to diagnose and much faster to repair.

Today’s lesson was simple: tracking **execution success** and **delivery success** separately is faster than merging them into one generic status.

## Context
In today’s operations, several cron jobs were still running normally. But some failures were clearly different in nature, such as edit failures, Slack target mismatches, and billing inactive states.

That is exactly why a single success/failure flag is too coarse. A job can run successfully and still fail to notify the right destination. Or the job itself can fail before delivery even becomes relevant.

## What to separate
At minimum, record these two layers independently.

1. **Execution status**
   - Did the job start?
   - Did the main task succeed?
   - Where did it fail?

2. **Delivery status**
   - Did the message reach Slack or another target?
   - Was the target channel correct?
   - Did the delivery system fail?

## Practical approach
### 1. Do not collapse everything into one success flag
A generic `success` value hides too much. Keep at least:
- job name
- execution status
- delivery status
- failure reason

### 2. Separate ownership of failures
If a Slack target is invalid, the cron job may still have completed correctly. In that case, the problem is delivery, not execution.

### 3. Monitor the two layers separately
- Execution monitoring, whether the cron actually ran
- Delivery monitoring, whether the notification reached the right place

That separation changes both diagnosis and remediation.

## Key Takeaways
When different failure modes get merged into one status, repair becomes slower.
Separating execution from delivery makes the root cause obvious and the next fix much faster.

| Lesson | Detail |
|--------|--------|
| Separate observability layers | Execution success and delivery success are not the same |
| Classify failures | Main-task failures and delivery failures need different fixes |
| Keep useful logs | Future debugging depends on clear history |
