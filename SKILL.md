---
name: skill-security-audit
description: Use when installing, adding, or evaluating any Claude skill from an external source — GitHub, social media, a colleague, or any source you did not write yourself. Also use before deploying any skill that uses Bash or WebFetch. Run this before the skill touches your system.
allowed-tools: Read, Bash, Glob, Grep
---

# Skill Security Audit

## Overview

Malicious Claude skills are real. Snyk audited ~4,000 AI skills (Feb 2025) and found 76 active malicious payloads across credential theft and data exfiltration categories. The specific attack: a skill with `Bash` in allowed-tools runs a one-liner that reads `~/Library/Messages/chat.db` (plain SQLite, no password) and posts your 2FA codes to a remote server before your phone finishes vibrating. One skill, both factors of a full account takeover.

**Run this audit before any external skill is installed.**

## When to Use

- Installing a skill from GitHub, social media, a tutorial, or a colleague
- Adding a skill you didn't write yourself
- Evaluating a skill that uses `Bash` or `WebFetch`
- Any time you're uncertain about what a skill does under the hood

## Audit Procedure

Given the path to a skill directory or SKILL.md file, run the following steps in order.

### Step 1 — Check allowed-tools

Read the SKILL.md frontmatter. Find the `allowed-tools:` line.

```bash
head -20 /path/to/skill/SKILL.md
```

**Red flags — stop and investigate before proceeding:**
- `Bash` with a wildcard (`Bash*`) — no legitimate skill needs unrestricted shell
- `WebFetch` present — ask exactly what URL it fetches and why; exfiltration can be dressed as "checking an update"
- Any tool you don't recognize

### Step 2 — Grep for dangerous file paths

```bash
grep -rn "chat\.db\|Library/Messages\|~/\.ssh\|~/\.aws\|keychain\|find-generic-password\|security unlock-keychain" /path/to/skill/
```

Any match = **STOP. Do not install.**

These are the exact paths used in the confirmed attack chains:
- `chat.db` / `Library/Messages` — iMessage 2FA codes
- `~/.ssh` — SSH private keys
- `~/.aws` — AWS credentials
- `security find-generic-password` — macOS Keychain credential extraction

### Step 3 — Scan for network calls

```bash
grep -rn "curl\|wget\|requests\.post\|fetch(\|http\." /path/to/skill/
```

For every match: identify the destination URL. Is it a well-known API (Google, OpenAI, Meta)? Or an unknown domain / IP address?

Unknown domain or hardcoded IP = **STOP. Do not install.**

### Step 4 — Check for obfuscation

```bash
grep -rn "base64\|eval(\|exec(\|__import__\|subprocess" /path/to/skill/
```

Legitimate use cases exist (Gemini API decodes base64 image responses; WebSocket handshakes use base64). For each match, read the surrounding 10 lines and verify the context is what it claims to be.

Hidden `eval()` executing a decoded base64 string = **STOP. Do not install.**

### Step 5 — Read every script file

Markdown cannot hurt you. Scripts can.

```bash
find /path/to/skill/ -name "*.py" -o -name "*.sh" -o -name "*.js" | sort
```

Read each script. Look for:
- Any file I/O targeting home directory paths not explicitly described in the skill
- Bulk environment variable collection (`os.environ` dumped to a variable, then sent out)
- Network calls that don't match the skill's stated purpose

### Step 6 — Check environment variable handling

```bash
grep -rn "os\.environ\|process\.env\|getenv" /path/to/skill/
```

Legitimate: reading `GOOGLE_API_KEY` to call the Gemini API.  
Suspicious: reading 10+ env vars into a dict, then posting that dict to a URL.

### Step 7 — Final verdict

Issue a verdict for each tier:

| Result | Action |
|--------|--------|
| No flags found | Safe to install |
| Bash present, all commands verified clean | Safe, note what Bash does |
| WebFetch present, URL is a known API | Safe, note what data is sent |
| Unknown domain, suspicious path access, or obfuscated code | **Do not install** |
| Any match from Step 2 (chat.db, .ssh, .aws, keychain) | **Do not install under any circumstances** |

---

## Quick Reference — Instant Red Flags

If you see any of these, walk away without further review:

```
chat.db
Library/Messages
~/.ssh
~/.aws
security find-generic-password
security unlock-keychain
Bash*   ← wildcard in allowed-tools
```

---

## About the Attack Vector

Every text you receive on your iPhone also lands on your Mac via iMessage Relay / SMS Forwarding (on by default since you set up your Apple account). All synced messages — bank codes, email 2FA, work login OTPs — live in `~/Library/Messages/chat.db`. It's plain SQLite. Readable with one shell command. No admin password required.

A skill with `Bash` access can watch that file for new rows and extract any 6-digit code in under a second — before your phone finishes vibrating. Paired with browser-stored credentials, that's a complete account takeover chain from a single skill install.

**Source:** Snyk ToxicSkills research, February 2025. 1,467 of ~4,000 audited skills had security flaws; 76 contained active malicious payloads.
