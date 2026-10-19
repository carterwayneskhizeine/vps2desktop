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
- npm ≥ 12 blocks postinstall scripts — the install "succeeds" with only a warning and the CLI is broken until rerun with `--allow-scripts=@earendil-works/pi-coding-agent` (verified 2026-10-06).
- **The transitive packages need it too.** pi pulls `protobufjs` and
  `@google/genai`, each with its own postinstall; npm reports them separately
  from pi's own script, so `--allow-scripts=@earendil-works/pi-coding-agent`
  alone leaves those two blocked. Rerun with every package named:
  `npm install -g @earendil-works/pi-coding-agent@latest --allow-scripts=@earendil-works/pi-coding-agent --allow-scripts=@google/genai --allow-scripts=protobufjs`.
  Check for a clean install by the absence of `npm warn install-scripts` lines.
- An official installer also exists (`curl -fsSL https://pi.dev/install.sh | sh`); npm is used here for the single-runtime policy.
- pi's own config directory is `/root/.pi` — empty until first run, that is normal.

## Manual follow-ups

- Configure model/provider on first run.
