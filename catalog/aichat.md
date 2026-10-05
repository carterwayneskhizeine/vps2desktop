# aichat

| | |
|---|---|
| id | `aichat` |
| group | ai-cli |
| deps | none |
| channel | github-api (asset names carry the version — no stable latest URL) |

sigoden's aichat — all-in-one LLM CLI (Rust, static musl binary).

## Guard (idempotency)

```bash
test -x /root/.local/bin/aichat && echo present
```

## Install

```bash
cd /tmp
# Asset filenames embed the version tag (aichat-v0.30.0-...), so resolve the tag first
TAG=$(curl -fsSL https://api.github.com/repos/sigoden/aichat/releases/latest | grep -oP '"tag_name":\s*"\K[^"]+')
curl --fail --location --retry 3 -o aichat.tar.gz \
  "https://github.com/sigoden/aichat/releases/download/$TAG/aichat-$TAG-x86_64-unknown-linux-musl.tar.gz"
tar -xzf aichat.tar.gz
install -m 755 aichat /root/.local/bin/aichat
rm -rf aichat aichat.tar.gz
```

## Verify

```bash
/root/.local/bin/aichat --version
```

## Known Pitfalls

- There is **no** stable `releases/latest/download/...` URL for aichat (versioned asset names) — the API lookup above is the correct latest channel; do not hardcode any tag.
- The musl static binary runs on any x86_64 Linux — no libc concerns.

## Manual follow-ups

- aichat needs `/root/.config/aichat/config.yaml` (model + API key). Typical practice: copy the whole `~/.config/aichat/` directory from an existing machine you own. Never committed anywhere by this project.
