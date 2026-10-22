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
- **The config directory holds API keys.** `config.yaml` carries the `api_key` of
  every client it lists, and the `roles/` and `macros/` files may hold
  credentials of their own. Ship the directory as `700` and the files as
  `600`. A hand-over copy is often created by `tar` as root and therefore
  inherits the source's permissions — a source file found at `755` (readable
  by every local account) stays `755` after the copy. Re-own the result by
  name, not by the numeric uid: the same uid means a different account on
  another host.
- **`--version` does not read the config.** A host with no `~/.config/aichat/`
  at all still passes the `Verify` above, so the version check alone cannot
  prove a hand-over worked. Use the checks in *Manual follow-ups* instead.

## Manual follow-ups

- aichat needs `/root/.config/aichat/config.yaml` (model + API key). Typical practice: copy the whole `~/.config/aichat/` directory from an existing machine you own. Never committed anywhere by this project.
- The hand-over moves **the whole directory**, not just the YAML: `config.yaml`
  plus `functions/`, `macros/` and `roles/`. Copying only `config.yaml` loses
  the custom roles (`roles/*.md`) and macros (`macros/*.yaml`) — `aichat
  --list-roles` then shows only the built-in ones.
- After the copy, tighten the permissions (`chmod 700 ~/.config/aichat` and
  `chmod 600` on the files) and prove the config was actually picked up:
  ```bash
  aichat --list-models      # must list every client/model from config.yaml
  aichat --list-roles       # must include the copied custom roles
  aichat --info             # shows the effective default model
  aichat -S "reply with OK" # one real call proves the API key works
  ```
