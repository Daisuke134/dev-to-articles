---
title: "How I Discovered My AI App Builder Has No 50-Iteration Limit"
published: true
tags: ai, debugging, automation, claude
---

## TL;DR

My Mobile App Factory kept stopping around 50 iterations. After digging through logs, I found there's **no actual limit**. Claude Code completes 2+ tasks too efficiently → ralph.sh enforces "1 iteration = 1 task" rule → auto-resets extra completions → `prd.json` flags become unreliable → `progress.txt` (immutable log) is the true source of truth.

## The Mystery: Why 50 Iterations?

For weeks I wondered: **Why does my Factory always stop around iteration 50?**

Looking at Desk Stretch Timer's `prd.json`, I saw US-008b/c/d/e all marked `passes: false` with notes saying `[reset: 1 US per iteration rule]`.

```json
{
  "id": "US-008e",
  "title": "Preflight + TestFlight",
  "passes": false,
  "notes": "[reset: 1 US per iteration rule]"
}
```

But checking `progress.txt` (an append-log), US-008e was completed on 2026-03-06. Contradiction.

## Investigation: Three Sources

### 1. progress.txt (append-only log)

```
2026-03-06 US-008e GREENLIT (0 blocking)
2026-03-06 US-008e asc validate Errors=0
2026-03-06 US-008e TestFlight beta review WAITING_FOR_REVIEW
2026-03-06 US-008e Public link: https://testflight.apple.com/join/78sH7CE9
```

→ **US-008e is complete**

### 2. prd.json

```json
{
  "id": "US-008e",
  "passes": false
}
```

→ **Completion flag is false (contradiction)**

### 3. ralph.sh (control script)

Here's where I found the truth:

```bash
if NEW_COUNT > 1:
    echo "🔴 1 US enforcement violation: $NEW_COUNT US completed in 1 iteration"
    # Keep the first completion, reset the rest
    KEEP_FIRST=$(echo "$NEW_PASSES" | xargs | awk '{print $1}')
    # Reset remaining to passes: false
```

## Root Cause: 1 Iteration = 1 US Enforcement

Claude Code is too efficient. It **completes 2+ User Stories in a single iteration**.

ralph.sh enforces "1 iteration = 1 US" for quality control. Any extra completions get auto-reset to `passes: false`.

**Result:**
- `prd.json`: Auto-reset target → unreliable
- `progress.txt`: Immutable log → **true source of truth**

## Fix: Trust progress.txt

```bash
# Check progress this way
cat progress.txt | grep "US-008e"

# prd.json passes flag is just a reference
cat prd.json | jq '.userStories[] | select(.id=="US-008e")'
```

Actual Desk Stretch Timer status:
- US-001 ~ US-008e: **Complete** (confirmed via progress.txt)
- US-009 (App Privacy + submit): **Not started**

## Key Takeaways

| Lesson | Detail |
|--------|--------|
| **Immutable logs are truth** | Overwritable flags (`prd.json`) get auto-reset. Immutable logs (`progress.txt`) are the reliable completion record |
| **Efficiency has side effects** | Claude Code's speed → 1 iteration = 1 US enforcement → auto-resets → appears "stuck" |
| **No limit exists** | "50-iteration limit" was a misunderstanding. Real issue: quality control design (1 US enforcement) |

## Summary

There was no "max 50" limit. The real cause: Claude Code's efficiency × quality control's 1-iteration-per-US rule creating auto-reset behavior.

**Debugging takeaways:**
1. Read 3+ sources (prd.json, progress.txt, scripts)
2. Trust immutable logs over overwritable flags
3. Look for design intent behind "limits"

A quiet Sunday spent deep-diving past logs revealed system behavior I hadn't seen before.
