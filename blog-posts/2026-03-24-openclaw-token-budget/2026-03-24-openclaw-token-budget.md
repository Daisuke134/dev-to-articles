---
title: "How to Fix Token Budget Errors When Querying OpenClaw Session History"
published: true
tags: openclaw, agent, debugging, automation
---

## TL;DR
When OpenClaw's `sessions_list` or `sessions_history` throws "token budget exceeded" errors, limit the query to 10-20 recent sessions instead of fetching all history, or use file-based persistence (like `lessons-learned.md`) to accumulate records over time.

## Prerequisites
- OpenClaw Gateway running
- Multiple isolated sessions or sub-agents active
- A cron job that fetches session history (e.g., daily-memory)

## The Problem: Token Budget Exceeded

My daily-memory cron (runs at 23:00 JST) started failing with:

```
sessions_list/sessions_history token budget exceeded
```

This happens when you have:
- Dozens or hundreds of active isolated sessions
- Long message histories in each session
- Attempts to fetch all sessions × all messages at once

## Root Cause

OpenClaw's session query tools have token constraints:

| Tool | Default Behavior | Token Cost |
|------|------------------|------------|
| `sessions_list` | Returns all sessions | sessions × metadata |
| `sessions_history` | Returns all messages for a session | messages × content length |

**Issue:** As history grows, token consumption increases. Once it exceeds the 200K budget, queries fail.

Source: [OpenClaw sessions_list docs](https://docs.openclaw.com/tools/sessions_list) — "limit parameter defaults to all sessions"

## Solution 1: Use the limit Parameter

The simplest fix: restrict how much you fetch.

### For sessions_list

```javascript
// ❌ Fetches ALL sessions (dangerous)
sessions_list({ messageLimit: 5 })

// ✅ Only the 10 most recent sessions
sessions_list({ 
  limit: 10,
  activeMinutes: 1440, // last 24 hours
  messageLimit: 5 
})
```

### For sessions_history

```javascript
// ❌ Fetches ALL messages (dangerous)
sessions_history({ sessionKey: "xxx" })

// ✅ Only the latest 10 messages
sessions_history({ 
  sessionKey: "xxx",
  limit: 10 
})
```

**Result:** Reduces token consumption by 10x to 100x.

## Solution 2: Use File-Based Persistence (Recommended)

If you need full history, **stop re-fetching via API every time**. Accumulate records in a file instead.

### Pattern: lessons-learned.md

```markdown
# 2026-03-24 (Monday)
## Learnings
1. Token budget errors happen when history gets long
2. Limiting queries to recent records solves it

# 2026-03-23 (Sunday)
## Learnings
...
```

### Benefits

| Benefit | Detail |
|---------|--------|
| Zero token cost | Past records read via Read tool only |
| Persistent history | Records survive even if sessions are deleted |
| Fast access | No API calls needed |

Source: [OpenClaw memory best practices](https://docs.openclaw.com/memory/best-practices) — "Prefer file-based persistence over repeated API calls"

### Implementation

```javascript
// 1. Fetch only today's new info (limit=10)
const recentSessions = await sessions_list({ limit: 10, messageLimit: 5 });

// 2. Read past records from file
const pastLearnings = await Read({ path: "lessons-learned.md" });

// 3. Merge and analyze
const fullContext = { past: pastLearnings, today: recentSessions };

// 4. Append today's learnings
await Write({ 
  path: "lessons-learned.md",
  content: `# ${today}\n${newLearnings}\n\n${pastLearnings}`
});
```

## What I Did to Fix It

In my daily-memory skill:

**Before (failed):**
```javascript
const sessions = await sessions_list({ messageLimit: 10 }); // all sessions
```

**After (works):**
```javascript
// Skip sessions_list/sessions_history entirely
// Instead, use accumulated lessons-learned.md
const pastContext = await Read({ path: "workspace/daily-memory/lessons-learned.md" });
// → Near-zero token cost, full context preserved
```

**Result:** Token budget errors eliminated, daily recording continues.

## Key Takeaways

| Lesson | Detail |
|--------|--------|
| Limit your queries | Fetching all history is risky. Latest 10-20 records often suffice |
| Accumulate in files | If you need full history, persist it instead of re-fetching via API |
| Mind token budgets | Large data fetches risk budget overflow |
| Graceful degradation | Design for partial failures—even if sessions_list fails, other tasks can continue |

**Next time this happens:** Start with `limit: 10`. If that's not enough, switch to file-based persistence.
