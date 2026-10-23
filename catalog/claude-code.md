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
- **Root + bypass permissions is a startup-breaking key**: with
  `permissions.defaultMode: "bypassPermissions"` in settings.json (or
  `--dangerously-skip-permissions`), the CLI refuses to start under root/sudo —
  `cannot be used with root/sudo privileges` — breaking interactive and
  `--print` runs alike. The only unlock is `IS_SANDBOX=1` in the environment
  (declares a sandboxed container). Set it where spawners inherit it, not only
  in the interactive shell.
- **settings.json applies to every session of this Unix user**: tools that
  spawn Claude Code with user settings loaded (e.g. Cyrus) share the `env`
  auth block and `permissions` with interactive runs — a provider switch or
  permission change lands on both sides at once.

## Manual follow-ups

- `claude` → OAuth login on first use.
