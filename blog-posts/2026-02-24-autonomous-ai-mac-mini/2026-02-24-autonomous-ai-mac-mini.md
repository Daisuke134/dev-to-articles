---
title: "How to Achieve Zero-Intervention AI Agent Operations After Mac Mini Migration"
published: false
tags: devops, macos, ai, automation
---

## TL;DR
Migrated 43 cron jobs and AI agent operations from VPS to Mac Mini. Achieved 100% autonomous operations with zero manual interventions. This guide covers LaunchAgent setup, auto-recovery configuration, and monitoring automation.

## Prerequisites
- OpenClaw Gateway environment
- Mac Mini (macOS Sonoma 14.6+)
- Existing AI agent operations on VPS
- 40+ cron jobs requiring migration

## The Problem with VPS Operations

Before migration, our VPS environment faced recurring issues:

| Issue | Frequency | Impact |
|-------|-----------|---------|
| Disk space exhaustion | Monthly | Skill execution failures |
| Power outage recovery | 2-3x/year | Hours of downtime |
| Session management complexity | Daily | Manual intervention required |

## Step 1: LaunchAgent Configuration for Auto-Startup

```bash
# Create LaunchAgent directory
mkdir -p ~/Library/LaunchAgents

# Configure OpenClaw Gateway auto-startup
cat > ~/Library/LaunchAgents/com.openclaw.gateway.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.openclaw.gateway</string>
    <key>ProgramArguments</key>
    <array>
        <string>/opt/homebrew/bin/openclaw</string>
        <string>gateway</string>
        <string>start</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>WorkingDirectory</key>
    <string>/Users/anicca/.openclaw</string>
</dict>
</plist>
EOF

# Load the LaunchAgent
launchctl load ~/Library/LaunchAgents/com.openclaw.gateway.plist
```

## Step 2: Enable Auto-Login for Unattended Operation

```bash
# Configure automatic login (use carefully in production)
sudo defaults write /Library/Preferences/com.apple.loginwindow autoLoginUser "anicca"
```

**Security Note**: Auto-login should be used on dedicated machines with physical security controls.

## Step 3: Standardize Environment Variables and PATH

```bash
# Add essential configurations to ~/.zshrc
cat >> ~/.zshrc << 'EOF'
export PATH=/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin
export CLAUDE_CODE_OAUTH_TOKEN=sk-ant-oat01-...  # Your actual token
source ~/.openclaw/.env
EOF
```

## Step 4: Test Power Failure Recovery

```bash
# Simulate power failure
sudo shutdown -h now

# Post-reboot verification script
#!/bin/bash
echo "=== Auto-Recovery Verification ==="
ps aux | grep openclaw | grep -v grep
echo "Gateway PID: $(ps aux | grep openclaw | grep -v grep | awk '{print $2}')"
```

## Step 5: Automated Monitoring and Alerting

```bash
# Set up daily system status reports via Slack
# Cron example: Daily 6:00 AM system status to #metrics
0 6 * * * /opt/homebrew/bin/openclaw exec "Execute daily-memory skill"
```

## Results: Autonomous Operation Metrics

| Metric | Before (VPS) | After (Mac Mini) |
|--------|--------------|------------------|
| Manual interventions | 2-3x/week | 0x/month |
| Power outage recovery | 30-60min | 3-5min (auto) |
| Disk management | Manual cleanup | Auto-rotation |
| Session management | SSH required | Fully automated |

## 30-Day Post-Migration Performance

- **Uptime**: 99.8% (excluding planned maintenance)
- **Manual interventions**: 0
- **Auto-recovery success**: 100%
- **Cron job success rate**: 99.2%

## Critical Success Factors

The difference between partial automation and true autonomous operation lies in these details:

### 1. LaunchAgent vs. Other Methods
```bash
# ❌ Cron @reboot - unreliable on macOS
@reboot /path/to/script

# ✅ LaunchAgent - macOS native, reliable
<key>RunAtLoad</key><true/>
```

### 2. Environment Isolation
Keep system environment (PATH, tokens) separate from application config:
- System: `~/.zshrc`  
- Application: `~/.openclaw/.env`

### 3. Recovery Testing
Power failure testing revealed that without auto-login, cron jobs fail without notice. They require user context to function properly.

## Key Takeaways

| Lesson | Detail |
|--------|--------|
| LaunchAgent is essential | Guarantees reliable auto-startup on system reboot |
| Auto-login required | Cron jobs fail without user login context |
| Environment variable separation | Clear distinction between system and app configs |
| Power failure testing mandatory | Verify recovery procedures under unexpected shutdown |
| Automated monitoring crucial | Slack reporting enables hands-off operations |

## Business Impact

Achieving true autonomous operations freed up 5-8 hours/week. Time before spent on infrastructure management now goes to product development. For solo developers, this gain is transformative.

The system now handles Eastern European user growth and business expansion (MRR: $22, 3 paying users). It requires zero infrastructure attention. Proper automation setup scales with business growth.