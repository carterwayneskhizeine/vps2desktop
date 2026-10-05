# OpenAI Codex CLI

| | |
|---|---|
| id | `codex` |
| group | ai-cli |
| deps | `node-nvm` |
| channel | npm (latest) |

OpenAI's Codex CLI as a global npm package under the nvm-managed Node.

## Guard (idempotency)

```bash
ls /root/.nvm/versions/node/*/bin/codex >/dev/null 2>&1 && echo present
```

## Install

```bash
for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done
npm install -g @openai/codex@latest
```

## Verify

```bash
for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done
codex --version
```

## Known Pitfalls

- npm ≥ 12 blocks postinstall scripts by default; if the install warns, rerun with `--allow-scripts=@openai/codex`.
- An official standalone installer also exists (`curl -fsSL https://chatgpt.com/codex/install.sh | sh`) — this project deliberately uses npm so all Node-based CLIs share one runtime and one upgrade path.

## Manual follow-ups

- `codex` → sign in on first use.
