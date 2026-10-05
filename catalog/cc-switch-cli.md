# cc-switch CLI

| | |
|---|---|
| id | `cc-switch-cli` |
| group | ai-cli |
| deps | none |
| channel | github-latest (`releases/latest/download/`, verified pattern) |

Provider/config switcher for Claude Code, Codex, Gemini, OpenCode, Hermes, OpenClaw and pi — swap API providers, manage MCP servers, prompts and skills from one place.

## Guard (idempotency)

```bash
test -x /root/.local/bin/cc-switch && echo present
```

## Install

```bash
curl --fail --location --retry 3 -fsSL \
  https://github.com/SaladDay/cc-switch-cli/releases/latest/download/install.sh | bash
```

## Verify

```bash
/root/.local/bin/cc-switch --version
```

## Known Pitfalls

- The `releases/latest/download/install.sh` URL is the upstream-documented pattern and auto-tracks new releases — the repo's release assets keep stable names, so it keeps working.
- Install order does not matter, but it only manages tools that exist — run `cc-switch` after the AI CLIs it should manage are installed if you want live switching immediately.
