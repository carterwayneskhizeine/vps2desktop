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
ls /root/.nvm/versions/node/*/bin/agent-browser >/dev/null 2>&1 && test -d /root/.cache/agent-browser && echo present
```

## Install

```bash
export PATH=/root/.nvm/versions/node/*/bin:$PATH
export DEBIAN_FRONTEND=noninteractive
npm install -g agent-browser@latest
agent-browser install --with-deps   # downloads browser binaries + apt-installs their system deps
```

## Verify

```bash
export PATH=/root/.nvm/versions/node/*/bin:$PATH
agent-browser --version
```

## Known Pitfalls

- npm ≥ 12 blocks postinstall scripts; if the install warns, rerun with `--allow-scripts=agent-browser`.
- `install --with-deps` runs apt under the hood — that is expected, not an escape from the apt channel rules.
- Optional but useful for recording: `apt-get install -y ffmpeg` (user's choice; not auto-installed).
- The cache dir check in the Guard may need adjusting if upstream changes its browser cache location — verify with `agent-browser install` output if in doubt.
