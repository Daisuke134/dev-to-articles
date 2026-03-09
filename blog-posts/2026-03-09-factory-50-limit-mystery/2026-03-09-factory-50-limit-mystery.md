---
title: "How I Debugged Why Our Mobile App Factory Stops at 50 Iterations"
published: true
tags: ci, automation, debugging, claude
---

## TL;DR

Investigated why our Mobile App Factory seemed to stop at ~50 iterations. Discovered there's no "max 50 limit"—it's the "1 iteration = 1 User Story" enforcement rule. `prd.json`'s `passes` flags are unreliable due to auto-reset behavior. `progress.txt` (append-mode log) serves as the source of truth.

## Prerequisites

- Mobile App Factory: iOS app auto-generation system powered by Claude Code (CC)
- `ralph.sh`: Factory control script
- `prd.json`: Manages User Story (US) completion flags
- `progress.txt`: Append-mode completion log

## The Problem: Why Does Factory Stop at ~50?

For the past week, I noticed a contradiction: `prd.json` showed US-008b/c/d/e as `passes: false`, but their notes indicated completion.

**Symptom:**
```json
{
  "id": "US-008e",
  "passes": false,
  "notes": "[reset: 1 US per iteration rule]"
}
```

## Step 1: Check progress.txt

```bash
cat ~/anicca-project/mobile-apps/desk-stretch-timer/progress.txt
```

**Discovery:**
- US-001 through US-008e all have completion records
- US-008e (TestFlight submission) completed on 2026-03-06
- TestFlight public link already generated

## Step 2: Read ralph.sh Code

```bash
grep -A 10 "1 US enforcement" ~/anicca-project/mobile-apps/desk-stretch-timer/ralph.sh
```

**Discovery: 1 Iteration = 1 US Enforcement Rule**

```python
if NEW_COUNT > 1:
    echo "🔴 1 US enforcement violation: $NEW_COUNT US completed in 1 iteration"
    # Keep the first completed US, reset the rest
    KEEP_FIRST=$(echo "$NEW_PASSES" | xargs | awk '{print $1}')
    # Reset remaining to passes: false
```

## Step 3: Check CC Session Logs

```bash
tail -100 ~/anicca-project/mobile-apps/desk-stretch-timer/logs/ralph.log
```

**Discovery:**
- US-008e completion: CC passed Greenlight / `asc validate` / release-review
- TestFlight beta review submission succeeded (WAITING_FOR_REVIEW)

## Root Cause

**There is no "max 50 limit."**

CC is so efficient that it completes 2+ US in one iteration. `ralph.sh` enforces "1 iteration = 1 US" to maintain quality control, automatically resetting the extra completed US.

## Key Takeaways

| Lesson | Detail |
|--------|--------|
| **prd.json is unreliable** | Auto-reset target. `passes` flag is for Factory's internal state management |
| **progress.txt is the source of truth** | Append-mode log. Never deleted or overwritten |
| **1 US per iteration rule intent** | Safety mechanism to prevent CC runaway and ensure adequate validation time per US |
| **Debug from append-mode logs** | Prioritize complete chronological logs over state management files (JSON, etc.) |

## Conclusion

The "stops at 50" perception was a misunderstanding. CC was working at high speed, and ralph.sh was auto-adjusting to ensure quality.

**Next time you face this issue:**
1. Check append-mode log (`progress.txt`) first
2. Treat state management files (`prd.json`) as reference
3. Read control script (`ralph.sh`) rules
