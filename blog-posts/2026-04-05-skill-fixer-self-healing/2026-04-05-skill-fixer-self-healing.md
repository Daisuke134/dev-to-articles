---
title: "How to Auto-Fix Broken AI Agent Cron Jobs with an LLM-Powered Self-Healer"
published: true
tags: devops, ai, automation, llm
---

## TL;DR

When 28 of my 38 AI agent cron jobs started throwing `complex interpreter invocation` errors simultaneously, manual fixing wasn't an option. I built `skill-fixer` — a cron job that uses an LLM to detect, patch, and commit fixes to broken skills automatically. By next morning: 28 errors → 0.

## Prerequisites

- An AI agent framework with scheduled cron jobs (e.g., OpenClaw)
- Skills/scripts that the agent runs on a schedule
- Node.js / bun runtime

## The Problem: 38 Crons, 50% Failure Rate

Anicca is an autonomous AI agent running on a Mac Mini. It manages 38 cron jobs: trend collection, TikTok slideshow generation, app nudge delivery, and more.

On 2026-04-04, 28 jobs suddenly started failing with:

```
Error: complex interpreter invocation detected
  at ~/.openclaw/skills/trend-hunter/index.ts:42
```

Root cause: an OpenClaw version update made a specific `exec` call pattern incompatible. Same error, 28 different skill files.

Fixing 28 files manually would take hours. I needed a different approach.

## Step 1: Identify the Error Pattern

First, collect failed job logs and find the common pattern:

```bash
# List recently failed cron jobs
openclaw tasks --failed --limit 50 | grep "complex interpreter"
```

All 28 failures shared the same root cause — a specific invocation pattern in skill files.

## Step 2: Design the Self-Healing Cron

Instead of manual fixes, I created `skill-fixer`: a cron job that feeds broken skill files to an LLM and applies the returned patches.

Three design decisions:

| Decision | Choice | Why |
|----------|--------|-----|
| Trigger time | 22:50 JST daily | After all other crons finish — avoids conflicts |
| LLM input | SKILL.md content + error log | Minimal context = faster, cheaper, less hallucination risk |
| Output | Patched SKILL.md committed to git | Reviewable, reversible, auditable |

## Step 3: Implementation

```typescript
// ~/.openclaw/skills/skill-fixer/index.ts
import { readFileSync, writeFileSync } from 'fs';

async function fixSkill(skillName: string, errorLog: string) {
  const skillPath = `~/.openclaw/skills/${skillName}/SKILL.md`;
  const content = readFileSync(skillPath, 'utf-8');

  const prompt = `
    The following SKILL.md causes this error: "${errorLog}"
    Output the fixed version.
    IMPORTANT: Only fix the error. Do not change anything else.
    
    ${content}
  `;
  
  const fixed = await callLLM(prompt);
  writeFileSync(skillPath, fixed);
  
  console.log(`✅ Fixed: ${skillName}`);
}

const failedSkills = await getFailedSkills(); // from OpenClaw API
for (const skill of failedSkills) {
  await fixSkill(skill.name, skill.errorLog);
}
```

The critical constraint: **"Only fix the error. Do not change anything else."** Without this, LLMs tend to "improve" things you didn't ask them to touch.

## Step 4: Add Idempotency

Prevent the same skill from being patched twice:

```bash
# Add a marker after fixing
echo "<!-- skill-fixer: fixed $(date +%Y-%m-%d) -->" >> "$SKILL_PATH"
```

On the next run, check for this marker and skip already-fixed files.

## Results

| Metric | Before | After |
|--------|--------|-------|
| `complex interpreter` errors | 28 | 0 ✅ |
| Manual fix time (estimated) | 4 hours | 0 minutes |
| skill-fixer runtime | — | ~800 seconds |
| Next-day success rate | 26% | 50% |

The next morning (2026-04-05): zero `complex interpreter` errors. Content generation crons produced 13 successful posts across TikTok, Instagram, and YouTube.

## Key Takeaways

| Lesson | Detail |
|--------|--------|
| At scale, manual fixes don't work | 28 broken files = you need automation |
| Constrain LLM scope explicitly | "Fix only this file, change only this error" prevents unwanted drift |
| Schedule repair jobs last | Run after all other crons to avoid mid-flight conflicts |
| Idempotency is non-negotiable | Without it, you risk double-patching and introducing new bugs |

As AI agent systems grow — more crons, more skills, more complexity — you need a self-healing layer. `skill-fixer` is the first implementation of that layer for Anicca. The goal: zero human interventions for routine breakage.
