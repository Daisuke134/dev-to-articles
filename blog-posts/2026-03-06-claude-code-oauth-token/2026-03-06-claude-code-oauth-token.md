---
title: "How to Recover Claude Code OAuth Token in 30 Seconds"
published: true
tags: claudecode, anthropic, oauth, macos, devops
---

## TL;DR
When Claude Code fails over SSH, extract the OAuth token from macOS Keychain. Inject it into `CLAUDE_CODE_OAUTH_TOKEN`. This restores access in 30 seconds without breaking tmux sessions.

## Prerequisites
- macOS (Keychain access required)
- Claude Code CLI v2.1.45 or later installed
- SSH access to your Mac or working in a tmux session
- Ran `claude setup-token` to generate an OAuth token

## Problem: SSH Can't Access Keychain

Claude Code stores OAuth tokens in macOS Keychain (service name: `Claude Code-credentials`). When running over SSH, Keychain is locked and the token is inaccessible.

```bash
$ echo "test" | claude -p
Error: Authentication required. Run 'claude auth login'
```

Running `claude auth login` opens a browser, which doesn't work over SSH.

## Solution: Pass the Token via Environment Variable

Claude Code prioritizes the `CLAUDE_CODE_OAUTH_TOKEN` environment variable over Keychain. Extract the token from Keychain and set it as an environment variable to make SSH access work.

## Step 1: Extract Token from Keychain

On a local Mac terminal (not via SSH), run:

```bash
security find-generic-password -s 'Claude Code-credentials' -w
```

Output example (JSON format):
```json
{"accessToken":"sk-ant-oat01-qCV5O13G...bcGbVQAA","refreshToken":"...","expiresAt":"2027-02-18T07:00:00.000Z"}
```

Copy the entire JSON output.

## Step 2: Set Environment Variable

If connected via SSH, set the environment variable:

```bash
export CLAUDE_CODE_OAUTH_TOKEN='{"accessToken":"sk-ant-oat01-qCV5O13G...bcGbVQAA","refreshToken":"...","expiresAt":"2027-02-18T07:00:00.000Z"}'
```

## Step 3: Inject into tmux Session (if using tmux)

If Claude Code is running in a tmux session, update the tmux environment:

```bash
tmux set-environment -t mobileapp-factory CLAUDE_CODE_OAUTH_TOKEN '{"accessToken":"sk-ant-oat01-qCV5O13G...bcGbVQAA","refreshToken":"...","expiresAt":"2027-02-18T07:00:00.000Z"}'
```

Load the environment variable inside the tmux session:

```bash
export CLAUDE_CODE_OAUTH_TOKEN=$(tmux show-environment -t mobileapp-factory CLAUDE_CODE_OAUTH_TOKEN | cut -d= -f2-)
```

## Step 4: Verify

```bash
echo "test prompt" | claude -p --allowedTools Bash,Read,Write
```

If it works, you're done.

## Persistence: Save to `.zshrc` and `.openclaw/.env`

To avoid repeating this every time, save the token to configuration files:

### Mac Mini
```bash
# Add to ~/.zshrc
export CLAUDE_CODE_OAUTH_TOKEN='{"accessToken":"sk-ant-oat01-qCV5O13G...bcGbVQAA","refreshToken":"...","expiresAt":"2027-02-18T07:00:00.000Z"}'
```

### OpenClaw Gateway
```bash
# Add to ~/.openclaw/.env
CLAUDE_CODE_OAUTH_TOKEN='{"accessToken":"sk-ant-oat01-qCV5O13G...bcGbVQAA","refreshToken":"...","expiresAt":"2027-02-18T07:00:00.000Z"}'
```

OpenClaw Gateway automatically loads `.openclaw/.env` on startup.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `security: SecKeychainSearchCopyNext: The specified item could not be found in the keychain.` | Never ran `claude setup-token` | Run `claude setup-token` in a local terminal and authenticate via browser |
| `Error: Authentication required` | Environment variable not set | Re-run Step 2 |
| Environment variable not reflected in tmux | tmux environment not updated | Run Step 3 |
| Token expired | `expiresAt` is in the past | Re-run Step 1 (Keychain stores the latest token) |

## Key Takeaways

| Lesson | Detail |
|--------|--------|
| **Keychain is inaccessible over SSH** | Claude Code tokens are stored in macOS Keychain, which is locked over SSH |
| **Environment variable solves it** | Setting `CLAUDE_CODE_OAUTH_TOKEN` makes SSH / tmux access work |
| **Persistence is key** | Save to `.zshrc` or `.openclaw/.env` to avoid repeating setup |
| **30-second recovery** | Extract from Keychain → set env var → inject into tmux = instant recovery |
| **No need to kill running processes** | Environment variable injection restores access without stopping tmux sessions |

This procedure restored access after a token expired in the mobileapp-factory-daily cron job. Recovery took 30 seconds. Ralph.sh execution resumed without interruption.
