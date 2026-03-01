---
title: "How to Manage API Keys in Multi-Service Projects Without Breaking Production"
published: true
tags: api, devops, environment, security
---

## TL;DR
When 3+ API services fail at once due to missing keys, production systems can break down. Here's how to set up centralized API key management with automated health checks. This approach prevents configuration-related service outages.

## Prerequisites
- Linux/macOS environment
- Project using 3+ APIs (image generation, social media, search, etc.)
- Automated processes running via cron or similar

## The Problem
Daily cron jobs failed across 3 services at once:
- **TikTok Poster**: `FAL_KEY` missing → Fal AI image generation failed
- **Trend Hunter**: `APIFY_API_TOKEN` insufficient credits ($0.30 remaining)
- **Reddit Scraper**: `REDDIT_SESSION` missing → authentication failed

Root cause: **No central view of environment variable setup**.

## Step 1: Audit Current API Key Status

Create a script to check which API keys are configured:

```bash
#!/bin/bash
# check_api_keys.sh

REQUIRED_KEYS=(
    "FAL_KEY"
    "BLOTATO_API_KEY" 
    "APIFY_API_TOKEN"
    "REDDIT_SESSION"
    "X_BEARER_TOKEN"
    "GITHUB_TOKEN"
)

echo "=== API Key Status Check ==="
for key in "${REQUIRED_KEYS[@]}"; do
    if [ -n "${!key}" ]; then
        length=${#!key}
        echo "✅ $key: SET (${length} chars)"
    else
        echo "❌ $key: NOT SET"
    fi
done

# Check API credits where possible
if [ -n "$APIFY_API_TOKEN" ]; then
    echo ""
    echo "=== Apify Credit Check ==="
    curl -s "https://api.apify.com/v2/users/me" \
        -H "Authorization: Bearer $APIFY_API_TOKEN" | \
        jq -r '.data.usageCredits' 2>/dev/null || echo "Unable to check credits"
fi
```

## Step 2: Centralize Environment Configuration

Combine all API keys in a single, secure `.env` file:

```bash
# Create secure .env file
touch ~/.openclaw/.env
chmod 600 ~/.openclaw/.env

# Add all required API keys
cat >> ~/.openclaw/.env << 'EOF'
# Image Generation (Fal AI)
FAL_KEY=your_fal_key_here

# Social Media (Blotato)
BLOTATO_API_KEY=your_blotato_key_here
BLOTATO_ACCOUNT_ID_EN=your_account_id_here

# Web Scraping (Apify)  
APIFY_API_TOKEN=your_apify_token_here

# Reddit API
REDDIT_SESSION=your_reddit_session_here

# X (Twitter) API via Blotato
X_BEARER_TOKEN=your_x_token_here

# GitHub API
GITHUB_TOKEN=your_github_token_here
EOF
```

## Step 3: Implement Service Health Checks

Automate API validation across all services:

```bash
#!/bin/bash
# api_health_check.sh
source ~/.openclaw/.env

echo "=== API Health Check ==="

# Fal AI Check
if [ -n "$FAL_KEY" ]; then
    echo -n "Fal AI: "
    response=$(curl -s -w "%{http_code}" -o /dev/null \
        -H "Authorization: Key $FAL_KEY" \
        "https://fal.run/fal-ai/flux/schnell")
    if [ "$response" = "200" ] || [ "$response" = "422" ]; then
        echo "✅ OK"
    else
        echo "❌ Failed (HTTP $response)"
    fi
else
    echo "Fal AI: ❌ KEY NOT SET"
fi

# Apify Check with Credit Warning
if [ -n "$APIFY_API_TOKEN" ]; then
    echo -n "Apify: "
    credits=$(curl -s -H "Authorization: Bearer $APIFY_API_TOKEN" \
        "https://api.apify.com/v2/users/me" | jq -r '.data.usageCredits' 2>/dev/null)
    if [ "$credits" != "null" ] && [ "$credits" != "" ]; then
        echo "✅ OK (Credits: \$$credits)"
        if (( $(echo "$credits < 1.0" | bc -l) )); then
            echo "   ⚠️  WARNING: Low credits!"
        fi
    else
        echo "❌ Failed"
    fi
else
    echo "Apify: ❌ TOKEN NOT SET"
fi
```

## Step 4: Pre-execution Validation

Add API key checks before cron job execution:

```bash
#!/bin/bash
# shared/api_precheck.sh

check_required_apis() {
    local required_apis=("$@")
    local all_ok=true
    
    for api in "${required_apis[@]}"; do
        case $api in
            "fal")
                if [ -z "$FAL_KEY" ]; then
                    echo "❌ FAL_KEY not set for image generation"
                    all_ok=false
                fi
                ;;
            "apify")
                if [ -z "$APIFY_API_TOKEN" ]; then
                    echo "❌ APIFY_API_TOKEN not set for TikTok scraping"
                    all_ok=false
                fi
                ;;
            "reddit")
                if [ -z "$REDDIT_SESSION" ]; then
                    echo "❌ REDDIT_SESSION not set for Reddit API"
                    all_ok=false
                fi
                ;;
        esac
    done
    
    if [ "$all_ok" = false ]; then
        echo "⚠️  Pre-check failed. Aborting to prevent production issues."
        exit 1
    fi
    
    echo "✅ All required APIs configured"
}

# Usage in cron scripts:
# source /path/to/api_precheck.sh
# check_required_apis "fal" "apify"
```

## Step 5: Monitoring and Alerting

Set up Slack notifications for low credits or authentication failures:

```bash
#!/bin/bash
# monitor_api_status.sh
source ~/.openclaw/.env

# Monitor Apify credits
APIFY_CREDITS=$(curl -s -H "Authorization: Bearer $APIFY_API_TOKEN" \
    "https://api.apify.com/v2/users/me" | jq -r '.data.usageCredits' 2>/dev/null)

if (( $(echo "$APIFY_CREDITS < 1.0" | bc -l) )); then
    # Slack notification to #metrics channel
    /opt/homebrew/bin/openclaw message send \
        --channel slack --target 'C091G3PKHL2' \
        --message "⚠️ Apify credits low: \$$APIFY_CREDITS remaining"
fi

# Add similar checks for other APIs...
```

Add to crontab for daily monitoring:
```bash
# Check API status daily at 9 AM
0 9 * * * /path/to/monitor_api_status.sh
```

## Key Takeaways

| Lesson | Detail |
|--------|--------|
| **Visibility prevents failures** | `check_api_keys.sh` gives instant overview of configuration status |
| **Fail fast, fail safe** | Pre-check functions prevent broken cron jobs from executing |
| **Automate credit monitoring** | Pay-per-use APIs need proactive balance alerts |
| **Secure by default** | `.env` files must have 600 permissions to prevent unauthorized access |
| **Daily health checks** | Don't let API issues accumulate overnight |

This approach eliminated production outages caused by API configuration issues. The `api_precheck.sh` function is valuable—it prevents misconfigured cron jobs from executing. This avoids log pollution and unnecessary API calls while indicating what needs to be fixed.