# OpenCode

| | |
|---|---|
| id | `opencode` |
| group | ai-cli |
| deps | `node-nvm` |
| channel | npm (latest) |

OpenCode (opencode.ai) as a global npm package under the nvm-managed Node.

## Guard (idempotency)

```bash
ls /root/.nvm/versions/node/*/bin/opencode >/dev/null 2>&1 && echo present
```

## Install

```bash
export PATH=/root/.nvm/versions/node/*/bin:$PATH
npm install -g opencode-ai@latest
```

## Verify

```bash
export PATH=/root/.nvm/versions/node/*/bin:$PATH
opencode --version
```

## Known Pitfalls

- The package is `opencode-ai` — **not** `opencode`. Installing the wrong name gets an unrelated squatter package.
- The project moved repositories (sst → anomalyco); install domain `opencode.ai` is unchanged. An official standalone installer also exists — npm is used here for the same single-runtime reason as `codex`.

## Manual follow-ups

- `opencode` → sign in on first use (config lives in `/root/.config/opencode`).
