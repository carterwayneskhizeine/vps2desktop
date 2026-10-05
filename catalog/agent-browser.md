# agent-browser

| | |
|---|---|
| id | `agent-browser` |
| group | ai-cli |
| deps | `node-nvm` |
| channel | npm (latest) + its own browser bootstrap |

Browser automation CLI for AI agents, as a global npm package plus its managed browser binaries.

## Guard (idempotency)

```bash
ls /root/.nvm/versions/node/*/bin/agent-browser >/dev/null 2>&1 && { test -d /root/.cache/agent-browser || test -d /root/.agent-browser/browsers; } && echo present
```

## Install

```bash
for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done
export DEBIAN_FRONTEND=noninteractive
npm install -g agent-browser@latest
agent-browser install --with-deps   # downloads browser binaries + apt-installs their system deps
```

## Verify

```bash
for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done
agent-browser --version
```

## Known Pitfalls

- npm ≥ 12 blocks postinstall scripts; if the install warns, rerun with `--allow-scripts=agent-browser`.
- `install --with-deps` runs apt under the hood — that is expected, not an escape from the apt channel rules.
- Optional but useful for recording: `apt-get install -y ffmpeg` (user's choice; not auto-installed).
- Browser storage differs by version: older builds download into `~/.cache/agent-browser`, 0.38+ into `~/.agent-browser/browsers` (verified on a live box). The Guard accepts either; if upstream moves again, check `agent-browser install` output.
