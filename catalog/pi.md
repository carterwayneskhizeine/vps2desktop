# pi coding agent

| | |
|---|---|
| id | `pi` |
| group | ai-cli |
| deps | `node-nvm` |
| channel | npm (latest) |

pi — the minimal, extensible coding agent (by Mario Zechner / earendil-works) as a global npm package.

## Guard (idempotency)

```bash
ls /root/.nvm/versions/node/*/bin/pi >/dev/null 2>&1 && echo present
```

## Install

```bash
for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done
npm install -g @earendil-works/pi-coding-agent@latest
```

## Verify

```bash
for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done
pi --version
```

## Known Pitfalls

- **Do not use `@mariozechner/pi`** — the project moved from badlogic to earendil-works; the current package is `@earendil-works/pi-coding-agent` (needs Node ≥ 22, satisfied by the LTS from `node-nvm`).
- An official installer also exists (`curl -fsSL https://pi.dev/install.sh | sh`); npm is used here for the single-runtime policy.
- pi's own config directory is `/root/.pi` — empty until first run, that is normal.

## Manual follow-ups

- Configure model/provider on first run.
