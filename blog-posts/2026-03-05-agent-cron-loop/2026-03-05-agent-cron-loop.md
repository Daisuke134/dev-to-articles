---
title: "How to Prevent AI Agent Cron Jobs from Silently Looping Forever"
published: true
tags: ai, devops, agentops, cron
---

## TL;DR

When you give an AI agent a recurring task via cron, it can loop the same check forever without explicit exit conditions. I caught my agent calling the RevenueCat API 30+ times in one heartbeat session — same result every time. It burned 10x the normal tokens. The fix: explicit completion criteria, iteration caps, and idempotency checks.

## The Bug: A Heartbeat That Wouldn't Stop

I run an autonomous AI agent (OpenClaw) with a heartbeat task that checks business metrics periodically. The task definition looked like this:

```markdown
## Revenue Watch
- Check MRR via RevenueCat API
- Alert on Slack if anomalies detected
```

Simple enough. Except the agent called the RevenueCat API **over 30 times** in a single session. Every call returned MRR $28, 5 subscribers. The agent reported the number, decided "next step: check Revenue Watch," and looped back.

**Root cause: no exit condition.** The task said *what* to do but never said *when to stop*.

## Why Agents Loop

| Cause | Detail |
|-------|--------|
| Missing completion criteria | "Check X" without "stop after checking" |
| No state persistence | Agent doesn't remember it already checked 5 seconds ago |
| Implicit continuation | Most agent frameworks keep running until the task list is empty |

Humans understand "check once and move on." Agents don't. They follow instructions to the letter.

## Step 1: Define Explicit Exit Conditions

```markdown
## Revenue Watch
- Check MRR via RevenueCat API
- **Exit: After one successful API response, log the result and stop**
- Only alert Slack if values changed from last check
```

The key addition: a clear statement of when the task is **done**.

## Step 2: Set Iteration Caps

Add a hard limit on loop iterations for each task.

```markdown
## Revenue Watch
- max_iterations: 1
- Fetch MRR → log → done
```

At the framework level:

```python
# LangChain example
agent = AgentExecutor(
    agent=agent,
    tools=tools,
    max_iterations=5,
    early_stopping_method="force"
)
```

This is your safety net. Even if the completion criteria has a bug, the agent stops after N iterations.

## Step 3: Add Idempotency Checks

Store the previous result and skip execution if nothing changed.

```markdown
## Revenue Watch
- Previous result: {cached_result}
- If current == previous: report "no change" and exit immediately
- If changed: report delta and update cache
```

This eliminates redundant API calls and makes the task naturally convergent.

## Step 4: System-Level Timeout

```bash
# Force-kill after 60 seconds
timeout 60 openclaw heartbeat run
```

This is the last line of defense. If everything else fails, the OS kills the process.

## Results After Fix

| Metric | Before | After |
|--------|--------|-------|
| API calls per execution | 30+ | 1 |
| Token consumption | 10x normal | 1x normal |
| Execution time | Minutes (looping) | Under 30 seconds |
| Accuracy | Repeated same data | Reports only on changes |

## Key Takeaways

| Lesson | Detail |
|--------|--------|
| Always define when a task ends | Agents don't infer "once is enough" |
| max_iterations is non-negotiable | Safety net for buggy exit conditions |
| Idempotency eliminates waste | Same input → skip, don't re-run |
| OS-level timeout is the last resort | Catches everything else |

When designing cron jobs for AI agents, ask "when should it stop?" — not "what should it do?" Without that answer, your agent will keep calling the same API endpoint forever.
