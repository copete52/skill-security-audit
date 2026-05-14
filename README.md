# skill-security-audit

A Claude Code skill that audits other Claude skills for malicious patterns before you install them.

## Why this exists

Snyk audited ~4,000 AI agent skills in February 2025 and found 76 active malicious payloads. The confirmed attack vector is simple: a skill with `Bash` in its allowed-tools runs a one-liner that reads `~/Library/Messages/chat.db` — plain SQLite, no password required — and ships your 2FA codes to a remote server. Every text you receive on your iPhone also syncs to your Mac via iMessage Relay. One skill install is the full account takeover chain.

## What it does

Runs a 7-step checklist against any skill directory:

1. **Check allowed-tools** — flags `Bash*` (wildcard) and unexpected `WebFetch`
2. **Grep for dangerous paths** — `chat.db`, `Library/Messages`, `~/.ssh`, `~/.aws`, keychain commands
3. **Scan for network calls** — `curl`, `wget`, `requests.post`, `fetch()` — verifies every destination
4. **Check for obfuscation** — `base64`, `eval()`, `exec()`, dynamic imports
5. **Read every script file** — `.py`, `.sh`, `.js` — markdown can't hurt you, scripts can
6. **Check environment variable handling** — reading one API key is fine; dumping all env vars to a URL is not
7. **Issue a verdict** — clean / note what Bash does / do not install

## Install

Copy the skill to your Claude skills directory:

```bash
cp -r skill-security-audit ~/.claude/skills/
```

Or clone directly:

```bash
git clone https://github.com/copete52/skill-security-audit ~/.claude/skills/skill-security-audit
```

## Usage

In any Claude Code session, when you're about to add an external skill:

```
/skill-security-audit
```

Then provide the path to the skill directory when prompted. Claude will run all 7 steps and return a verdict.

## Instant red flags

If you see any of these in a skill, don't install it — no further review needed:

```
chat.db
Library/Messages
~/.ssh
~/.aws
security find-generic-password
security unlock-keychain
Bash*   ← wildcard in allowed-tools
```

## License

MIT
