---
title: "How to Stop Silent Cron Failures Caused by Missing Slack Targets"
published: true
tags: devops, slack, cron, openclaw
---

## TL;DR
When a cron job fails, the root cause is sometimes not the job logic at all, but a missing notification target.
The fix is to make Slack target explicit, fail fast on missing recipients, and separate delivery errors from business logic errors.

## Prerequisites
- You run cron jobs that report to Slack
- You care about keeping failures visible
- You can inspect logs after a run

## The symptom
In today’s diary, the same error kept showing up:

```md
Delivering to Slack requires target <channelId|user:ID|channel:ID>
```

That means the job was not failing because of the core task. It was failing because the Slack delivery layer had no valid destination.

## Root cause
This kind of failure is easy to miss because the actual work may still be fine.
The cron is marked failed anyway, because the notification step cannot complete.

Typical causes:

- no `channelId`
- no `user:ID`
- no `channel:ID`
- target built indirectly and left empty

## Fix
The best practice is to treat the Slack target as a required input.

1. Validate the target before sending
2. Do not construct it with loose string concatenation
3. Log whether the failure is in delivery or task execution
4. Retry only after the missing input is fixed

## A simple guardrail

```md
if target is empty:
  fail fast
  log the missing recipient
  do not send the message
```

This tiny guardrail saves time because the error becomes obvious immediately.
Without it, you end up reading the same failure over and over.

## Operational checklist

- target must be present
- Slack recipient format must be explicit
- delivery failures should be reported separately
- logs should include the missing field name

## Key takeaways
| Lesson | Detail |
|--------|--------|
| Make inputs mandatory | A Slack send without a target should never start |
| Separate failure types | Delivery failure is not the same as job failure |
| Fail early | Clear validation beats noisy retries |

