# Node.js via nvm

| | |
|---|---|
| id | `node-nvm` |
| group | runtime |
| deps | none |
| channel | official-script (latest nvm release) + `nvm install --lts` + npm latest |

Latest nvm release (tag resolved via GitHub API, never pinned), latest Node **LTS** (satisfies every AI CLI's Node ≥ 22 requirement), and latest pnpm.

## Guard (idempotency)

```bash
ls -d /root/.nvm/versions/node/*/bin/node >/dev/null 2>&1 && echo present
```

## Install

```bash
# Latest nvm release tag (assets/branch URLs change per release; resolve via API)
NVM_TAG=$(curl -fsSL https://api.github.com/repos/nvm-sh/nvm/releases/latest | grep -oP '"tag_name":\s*"\K[^"]+')
curl --fail --location --retry 3 -o- "https://raw.githubusercontent.com/nvm-sh/nvm/$NVM_TAG/install.sh" | bash

# Non-interactive shells do NOT source .bashrc — drive nvm explicitly:
export NVM_DIR="/root/.nvm"
. "/root/.nvm/nvm.sh"
nvm install --lts
nvm alias default 'lts/*'

# The node bin dir must be on PATH *before* calling npm: `nvm install` does not
# export it into the calling shell, so the next line fails with
# "npm: command not found" (measured 2026-10-20 on a fresh Ubuntu 26.04).
for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done
export PATH
npm install -g npm@latest pnpm
```

## Verify

```bash
for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done
node -v && npm -v && pnpm -v
```

## Known Pitfalls

- **`nvm install` does not export the node bin dir into the calling shell.** The
  Install block used to call `npm install -g npm@latest pnpm` straight after
  `nvm install --lts`, which failed with `npm: command not found` — the newly
  installed Node was on disk but not on PATH. The block now puts the bin dir on
  PATH *before* the npm call. The same trap applies to every downstream npm
  component (codex, opencode, pi, agent-browser).
- **PATH discipline is everything here**: the nvm installer appends loader lines to `/root/.bashrc`, but that file early-exits for non-interactive shells — over SSH every install/verify command must put the node bin dir on PATH via `for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done` or use absolute paths. (A glob does NOT expand inside a `PATH=` assignment — a one-line `export PATH=…node/*/bin:$PATH` silently keeps the literal `*`.)
- Components depending on this one (`codex`, `opencode`, `pi`, `agent-browser`) resolve npm by absolute path: `/root/.nvm/versions/node/*/bin/npm`.
- `nvm install --lts` tracks the newest LTS line — do not substitute a hardcoded major version.
