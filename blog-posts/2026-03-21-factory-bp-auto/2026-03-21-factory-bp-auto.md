---
title: "How to Auto-Update Your PRD with Daily Best Practice Searches"
published: true
tags: automation, devops, bestpractices, productivity
---

## TL;DR

I built a system that updates our mobile app PRD (Product Requirements Document) every day by searching for best practices and adding them with full citations. Result: our development team (Claude Code) always has the latest optimization strategies, including a 636% LTV increase from weekly subscriptions + 7-day trials.

## Prerequisites

- Cron execution environment (or OpenClaw AI agent framework)
- Web search API (Brave Search, etc.)
- prd.json file (JSON-formatted PRD)
- Git repository

## The Problem: Stale PRDs Kill Optimization Opportunities

Mobile app development moves fast. Revenue optimization, UX patterns, and implementation efficiency best practices update daily. Manually updating docs is unrealistic:

| Problem | Impact |
|---------|--------|
| Implementing from old PRD | Miss 636% LTV opportunities |
| Search is ad-hoc | Inconsistent research depth |
| No citations | Can't verify, hallucination risk |

## Step 1: Design 3 BP Cron Jobs

I created three specialized crons:

```bash
# revenue cron (monetization BP)
schedule: cron 0 22 * * * (daily 22:00 JST)
payload: web_search for revenue BP → update prd.json

# efficiency cron (iOS dev BP)
schedule: cron 22 22 * * * (daily 22:22 JST)
payload: web_search for iOS dev BP → update prd.json

# internal cron (infrastructure validation)
schedule: cron 40 22 * * * (daily 22:40 JST)
payload: check .learnings/ERRORS.md → detect gaps
```

**Example search queries (revenue cron):**
- `mobile app subscription pricing LTV increase 2024 2025`
- `free trial conversion rate optimization study`
- `subscription plan comparison test results`

**Critical:** At least 3 keywords, English-first, year-specific for latest data.

## Step 2: Extract & Add BPs to prd.json

All BPs are recorded with 3-part citations (source + URL + core quote).

**prd.json update example (revenue cron results):**

```json
{
  "subscriptions": {
    "bestPractices": [
      {
        "title": "Weekly Subscription + 7-Day Trial = 636% LTV Increase",
        "source": "Mobile Growth Stack - Free Trial Length Study",
        "url": "https://www.mobilegrowthstack.com/free-trial-conversion-rate-benchmark/",
        "quote": "Top quartile apps (4+ day trials) see 60%+ trial-to-paid vs 38% average",
        "recommendation": "Switch to weekly plan + 7-day trial. Proven data: $7.40 monthly → $54.50 weekly (636% increase)"
      },
      {
        "title": "3-Plan Layout Increases Conversions by 44%",
        "source": "Reforge - Subscription Pricing Optimization",
        "url": "https://www.reforge.com/blog/subscription-pricing",
        "quote": "Decoy effect: 3 plans increase middle-tier selection by 44%",
        "recommendation": "Expand from 2 plans (current) to 3-plan layout (Basic/Premium/Pro)"
      }
    ]
  }
}
```

## Step 3: Implementation

**Automated search → update → commit:**

```javascript
// factory-bp-revenue.js (simplified)
async function updateRevenueBP() {
  // 1. Search for BPs
  const queries = [
    'mobile subscription pricing LTV 2025',
    'free trial conversion optimization',
    'paywall design best practices'
  ];
  
  const results = [];
  for (const q of queries) {
    const res = await webSearch(q, { count: 5 });
    results.push(...res);
  }
  
  // 2. Load prd.json
  const prd = JSON.parse(fs.readFileSync('prd.json', 'utf-8'));
  
  // 3. Add new BPs (dedup check)
  const newBPs = extractBestPractices(results);
  prd.subscriptions.bestPractices = [
    ...prd.subscriptions.bestPractices,
    ...newBPs.filter(bp => !isDuplicate(bp, prd))
  ];
  
  // 4. Save & commit
  fs.writeFileSync('prd.json', JSON.stringify(prd, null, 2));
  execSync('git add prd.json');
  execSync('git commit -m "chore: update revenue BP (auto)"');
  execSync('git push origin dev');
}
```

**Cron registration (OpenClaw example):**

```bash
openclaw cron add \
  --name factory-bp-revenue \
  --schedule '{"kind":"cron","expr":"0 22 * * *","tz":"Asia/Tokyo"}' \
  --payload '{"kind":"agentTurn","message":"Execute factory-bp-revenue skill"}' \
  --sessionTarget isolated
```

## Step 4: Results

**2026-03-21 execution results:**

| Cron | BPs Added | Key Findings |
|------|-----------|--------------|
| revenue | 12 BPs | Weekly+7d trial 636% LTV, 3-plan +44%, animation +12-18% |
| efficiency | 6 BPs | Never auto-edit .pbxproj (prevents 90% issues), iOS Simulator feedback loop |
| internal | 0 BPs | Detected missing `.learnings/ERRORS.md` (infra gap found) |

**Git commit verification:**
```bash
$ git log --oneline | head -3
6cdaffd chore: update revenue BP (auto)
bf91a6c chore: update efficiency BP (auto)
a3c8f12 chore: verify internal BP tracking
```

## Key Takeaways

| Lesson | Detail |
|--------|--------|
| **Automation value** | Capture 636% LTV insights you'd otherwise miss. Daily search is impossible for humans |
| **Citations mandatory** | Source+URL+quote = verifiable. No citations = hallucination risk |
| **3-cron separation** | revenue/efficiency/internal for clear responsibility, automated gap detection |
| **Git commit history** | BP additions are version-controlled, trackable when/what was added |
| **Search query strategy** | Min 3 keywords, English-first, year-specific for latest data |

**Next steps:**
- Build `.learnings/ERRORS.md` system for automated learning from repeated errors
- Track BP implementation (which PRD BPs actually got implemented)
- Weekly BP cleanup cron (auto-archive outdated BPs)

**GitHub:** [anicca-products](https://github.com/Daisuke134/anicca-products) (includes Mobile App Factory implementation)

