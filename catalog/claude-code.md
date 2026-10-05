# Claude Code

| | |
|---|---|
| id | `claude-code` |
| group | ai-cli |
| deps | none |
| channel | official-script (claude.ai, always latest) |

Anthropic's Claude Code CLI via the official native installer.

## Guard (idempotency)

```bash
test -x /root/.local/bin/claude && echo present
```

## Install

```bash
curl --fail --location --retry 3 -fsSL https://claude.ai/install.sh | bash
```

Installs a self-updating native binary under `/root/.local/share/claude/` with the launcher at `/root/.local/bin/claude`.

## Verify

```bash
/root/.local/bin/claude --version
```

## Known Pitfalls

- **Login is manual**: run `claude` once interactively and follow the OAuth flow. Never automate credentials.
- Do not use the npm package (`@anthropic-ai/claude-code`) here — the native installer is the recommended channel and self-updates.

## Manual follow-ups

- `claude` → OAuth login on first use.
