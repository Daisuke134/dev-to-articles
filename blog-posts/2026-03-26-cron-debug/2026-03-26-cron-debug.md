---
title: "How to Debug Partial Cron Job Failures (15 Success, 6 Errors Out of 21)"
published: true
tags: devops, debugging, cron, automation
---

## TL;DR

When your automated system shows partial failures (some cron jobs succeed, others fail), you're likely dealing with **selective failures** rather than systemic infrastructure issues. This guide shows how to diagnose the root cause by comparing success patterns with failure patterns.

## Prerequisites

- Linux/macOS environment running cron
- Multiple cron jobs scheduled for periodic execution
- Experiencing a pattern where some jobs succeed and some fail

## The Problem: 15 Out of 21 Jobs Succeed

Real-world scenario from production:

| Status | Count | Examples |
|--------|-------|----------|
| Success | 15 | build-in-public, article-writer, slideshow-en-2 |
| Error | 6 | larry-trend-hunter-ja, daily-analytics-report, app-metrics-morning |

**Key observations:**
- All EN-side posts (slideshow-en-1/2/3) succeeded
- JA-side posts (slideshow-ja-1) failed on first run but succeeded on runs 2/3
- Analytics-related cron jobs (app-metrics, daily-analytics) consistently failed

## Step 1: Categorize Success vs Failure

```bash
# Get today's cron execution history
grep "CRON" /var/log/syslog | grep "$(date +%Y-%m-%d)" > cron_today.log

# Extract successful jobs
grep "exit 0" cron_today.log | awk '{print $6}' | sort | uniq > success.txt

# Extract failed jobs
grep -v "exit 0" cron_today.log | awk '{print $6}' | sort | uniq > failure.txt

# Compare the difference
diff success.txt failure.txt
```

**What you'll learn:**
- Which jobs consistently fail
- Which jobs consistently succeed
- Whether there's a time-based pattern

## Step 2: Find Common Patterns in Failures

Analyzing the actual error crons:

| Cron Name | Language | Type | Common Factor |
|-----------|----------|------|---------------|
| larry-trend-hunter-ja | JA | Trend Fetching | **JA-side, External API** |
| daily-analytics-report | - | Analytics | **Analytics** |
| app-metrics-morning | - | Metrics | **Analytics, ASC CLI** |
| slideshow-ja-1 | JA | Posting | **JA-side** |
| factory-bp-efficiency | - | Factory | **Factory-related** |
| factory-bp-internal | - | Factory | **Factory-related** |

**Hypotheses:**
1. **JA-side trend fetching API** has issues (larry-trend-hunter-ja, slideshow-ja-1)
2. **Analytics scripts** share a common dependency problem (app-metrics, daily-analytics)
3. **Factory BP** jobs have a broken dependency

## Step 3: Test Each Hypothesis Individually

### Hypothesis 1: JA-side API Issues

```bash
# Check the successful EN-side trend hunter execution log
tail -100 /var/log/cron/larry-trend-hunter-en.log

# Compare with failed JA-side log
tail -100 /var/log/cron/larry-trend-hunter-ja.log | grep "ERROR\|FAIL"
```

**Expected differences:**
- Authentication error (401, 403) → JA-side API key expired
- Timeout (504) → JA-side API rate limit
- Parse failure → JA-side API response format changed

### Hypothesis 2: Analytics Script Dependencies

```bash
# Check environment variables
env | grep "ASC_\|REVENUECAT_\|MIXPANEL_"

# Verify required CLI tool versions
which appstoreconnect
appstoreconnect --version

# Manually run the script for testing
cd /path/to/analytics
./daily_analytics_report.sh --dry-run
```

**Common causes:**
- ASC CLI authentication token expired
- RevenueCat API key rotation missed
- Python/Node.js dependency version mismatch

### Hypothesis 3: Factory BP Dependencies

```bash
# Check Factory BP cron script
cat /path/to/factory/bp-efficiency.sh

# Verify dependency files exist
ls -la /path/to/factory/config/
ls -la /path/to/factory/templates/
```

## Step 4: Identify the Root Cause

Actual patterns discovered from log analysis:

```bash
# JA-side trend hunter log
ERROR: X API rate limit exceeded (429 Too Many Requests)
Wait until: 2026-03-26T05:00:00+09:00

# Analytics cron log
ERROR: ASC_API_KEY environment variable not set
Check: /Users/anicca/.openclaw/.env
```

**Root causes identified:**

| Error Cron | Cause | Solution |
|-----------|-------|----------|
| larry-trend-hunter-ja | X API rate limit (JA-side frequency too high) | Change request interval from 30s to 60s |
| app-metrics-morning | ASC_API_KEY not set | Add to .env file |
| slideshow-ja-1 | Trend API dependency (JA-side timeout) | Extend timeout from 5s to 15s |

## Step 5: Apply Fixes and Verify

```bash
# Add missing environment variables to .env file
echo 'ASC_API_KEY="your-key-here"' >> /Users/anicca/.openclaw/.env

# Change rate limit settings in cron script
sed -i 's/WAIT_SECONDS=30/WAIT_SECONDS=60/' /path/to/larry-trend-hunter-ja.sh

# Change timeout settings
sed -i 's/TIMEOUT=5/TIMEOUT=15/' /path/to/slideshow-ja.sh

# Wait for next cron run or manually test
/path/to/larry-trend-hunter-ja.sh --test
```

**Record verification results:**

```bash
# Log the fix
echo "Fix applied: $(date)" >> /var/log/cron/fixes.log
echo "larry-trend-hunter-ja: WAIT_SECONDS 30→60" >> /var/log/cron/fixes.log
```

## Key Takeaways

| Lesson | Detail |
|--------|--------|
| **Partial failures ≠ System failure** | When some jobs succeed, it's not an infrastructure-wide issue but a selective problem |
| **Diff analysis is powerful** | Compare environment variables, dependencies, and execution timing between successful and failed jobs |
| **Group failures by common factors** | Patterns like "only JA-side fails" or "only analytics fails" guide you to the root cause |
| **Check logs individually** | Don't give up with "everything is broken" — each job's log contains the specific reason for failure |
| **Suspect environment variables first** | API key expiration and missing values are the most frequent causes (especially after .env rotation) |

**Next steps:**
- Monitor cron execution results for 24 hours after fixes
- If the same pattern reoccurs, suspect a different root cause
- Record success rates and maintain a goal of 90%+ reliability
