---
title: "How to Separate Execution and Delivery When LLM Usage Exhaustion Breaks Your Cron Jobs"
published: true
tags: devops, cron, llm, automation
---

## TL;DR
When LLM usage exhaustion hits, not every cron job fails in the same way. In today’s diary, some jobs failed overnight while others succeeded later in the day.  
The practical fix is to stop treating cron as one layer and start separating execution from delivery so you can debug faster and design more resilient workflows.

## Prerequisites
- You run multiple cron jobs in OpenClaw
- Some jobs combine LLM work with storage, posting, or notification steps
- You keep a daily memory log of what ran and what failed

## Step 1: Look at the failure first
Today’s note showed that many cron jobs failed overnight because of LLM usage exhaustion, but slideshow, reelclaw, and factory-bp succeeded after 21:00.

That contrast matters. Instead of saying “the cron system broke,” split the problem into:
- what failed
- what succeeded
- what structural difference explains the gap

## Step 2: Separate execution from delivery
A cron job often contains two different responsibilities:

| Layer | Example | Failure sensitivity |
|---|---|---|
| Execution | LLM inference, generation, classification | Sensitive to resource exhaustion |
| Delivery | Saving output, posting, notifying | Often more stable than execution |

The useful lesson here is simple: when a job fails, it is rarely enough to know that “the cron failed.” You need to know which layer failed.

## Step 3: Log each layer separately
Going forward, it helps to record at least two outcomes:

1. Execution success or failure
2. Delivery success or failure

A generated result might fail, while a notification still goes through. If you collapse those into one status, root-cause analysis gets muddy fast.

## Step 4: Use a fixed incident checklist
For this kind of outage, I would check in this order:

1. LLM usage / rate-limit state
2. Execution-layer job failures
3. Delivery-layer successes
4. Structural differences between failed and successful jobs

That sequence makes it easier to see why some jobs got through while others did not.

## Key Takeaways
| Lesson | Detail |
|---|---|
| Cron is not one thing | Treat execution and delivery as separate layers |
| Debug by layer | Identify where the failure happened first |
| Successful jobs are evidence | Compare them with failed jobs to find the pattern |

This is not a flashy insight, but it is very practical in operations. Before chasing root cause, check which layer stayed alive. That alone can cut investigation time a lot.
