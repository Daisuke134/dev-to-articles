---
title: "How to Fix Authentication Token Mismatch in Multi-Service Deployments"
published: true
tags: devops, authentication, microservices, troubleshooting
---

## TL;DR
Authentication token mismatch between Railway, VPS, and Mac Mini caused partial API failures. Fixed by syncing INTERNAL_AUTH_SECRET and regenerating Gateway tokens. Separation of concerns kept core functions running despite visibility loss.

## Prerequisites
- Multi-environment microservice setup
- Shared authentication tokens between services  
- PaaS (Railway) + VPS + local environment architecture

## The Problem: Selective API Failures

```bash
# Symptoms observed
sessions_list → 403 Forbidden
app-nudge-evening → INTERNAL_AUTH_SECRET mismatch
75 skill executions → ✅ Working normally
```

**The key insight: Not everything failed at once.**

| API | Status | Root Cause |
|-----|--------|------------|
| sessions_list | ❌ 403 | Gateway token expired |
| app-nudge-evening | ❌ Auth failed | SECRET mismatch |
| Core skill execution | ✅ Normal | Auth-free or local |

## Root Cause: Token Sync Design Gaps

### Issue 1: Environment Variable Drift

```bash
# Railway environment
INTERNAL_AUTH_SECRET=abc123old

# Local environment  
INTERNAL_AUTH_SECRET=xyz789new
```

**Cause**: Manual Railway env update forgotten during local changes.

### Issue 2: Gateway Token Expiration  

```bash
# Symptom
openclaw status → Gateway token: expired
sessions_list → 403 Forbidden
```

**Cause**: Long-running system had token rotation, local config wasn't updated.

## Fix Steps

### Step 1: Verify SECRET Sync

```bash
# Check current values across environments
echo "Railway: $(railway env get INTERNAL_AUTH_SECRET)"
echo "Local: $INTERNAL_AUTH_SECRET"

# Sync to latest value if mismatch found
railway env set INTERNAL_AUTH_SECRET="$INTERNAL_AUTH_SECRET"
```

### Step 2: Regenerate Gateway Token

```bash
# Check current status
openclaw status
# → Gateway token status: expired

# Generate fresh token
openclaw gateway token-refresh
# → New token: gw_xxx...

# Update environment
export OPENCLAW_GATEWAY_TOKEN="gw_xxx..."
```

### Step 3: Verify Separation of Concerns

```bash
# Auth-required APIs (affected by tokens)
curl -H "Authorization: Bearer $TOKEN" api/sessions
curl -H "X-Internal-Secret: $SECRET" api/nudge

# Auth-free logic (unaffected)
local-skill-execution  # ✅ Continued working
file-operations        # ✅ Continued working
cron-jobs             # ✅ Continued working
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| sessions_list | ❌ 403 | ✅ Working |
| app-nudge-evening | ❌ Auth fail | ✅ Working |
| System automation | 78% (maintained) | 78% (maintained) |
| Core skills | 100% success | 100% success |

**Total fix time: 9 hours** (4h diagnosis + 3h root cause + 2h repair)

## Key Takeaways

| Lesson | Detail |
|--------|--------|
| **Design for partial failure** | Auth problems shouldn't kill core functionality |
| **Automate token synchronization** | Manual env updates always get missed |
| **Staged degradation over total failure** | Some APIs failing ≠ system down |
| **Visibility vs availability** | Can't see metrics ≠ system not working |

## Prevention Script

```bash
#!/bin/bash
# Token sync checker for cron
check_token_sync() {
    railway_secret=$(railway env get INTERNAL_AUTH_SECRET)
    local_secret=$INTERNAL_AUTH_SECRET
    
    if [ "$railway_secret" != "$local_secret" ]; then
        echo "🚨 Token mismatch detected"
        slack_alert "Auth tokens out of sync"
        exit 1
    fi
}

# Run every 6 hours
0 */6 * * * /path/to/check_token_sync.sh
```

Multi-environment auth will always drift. Don't rely on human memory—automate the checks and catch mismatches immediately.