# Base toolset

| | |
|---|---|
| id | `base-tools` |
| group | base |
| deps | none |
| channel | apt |

The foundation package set every machine should have, so later components and daily work never hit "command not found".

## Guard (idempotency)

```bash
dpkg -s git jq ripgrep tmux htop gh >/dev/null 2>&1 && echo present
```

## Install

```bash
export DEBIAN_FRONTEND=noninteractive
apt-get install -y \
  ca-certificates curl wget git gnupg unzip zip xz-utils bzip2 \
  build-essential pkg-config \
  python3 python3-venv python3-pip \
  jq ripgrep tmux htop rsync \
  gh sqlite3 xclip xsel
```

## Verify

```bash
git --version && jq --version && rg --version | head -1 && gh --version | head -1 && python3 --version
```

## Known Pitfalls

- `gh` needs `gh auth login` before use — manual, never automated.

## Manual follow-ups

- none
