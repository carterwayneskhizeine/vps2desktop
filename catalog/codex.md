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

## Permission posture (config.toml)

`~/.codex/config.toml` controls the CLI's sandbox and approvals. Top-level keys
must appear before any `[table]` header — appending one after a table silently
scopes it into that table.

```toml
sandbox_mode = "danger-full-access"       # read-only | workspace-write | danger-full-access
approval_policy = "never"                 # on-request | never

[sandbox_workspace_write]
writable_roots = ["/root/"]               # extra writable roots; workspace-write only
network_access = true                     # workspace-write only

shell_environment_policy.inherit = "all"  # pass the full environment through
tools.web_search = true
```

`codex doctor` prints the effective posture
(`filesystem unrestricted · network enabled` under `danger-full-access`).

## Known Pitfalls

- npm ≥ 12 blocks postinstall scripts by default; if the install warns, rerun with `--allow-scripts=@openai/codex`.
- An official standalone installer also exists (`curl -fsSL https://chatgpt.com/codex/install.sh | sh`) — this project deliberately uses npm so all Node-based CLIs share one runtime and one upgrade path.
- The ChatGPT desktop app co-manages `~/.codex/config.toml` (`[desktop]`,
  marketplaces, plugins, `mcp_servers.node_repl`); manual keys coexist, but
  re-check the file after app updates.
- Under `danger-full-access`, `writable_roots` and `network_access` are inert —
  there is no sandbox to configure. They take over when `sandbox_mode` returns
  to `workspace-write`.
- The beta `default_permissions` / `[permissions.<name>]` system cannot be
  combined with `sandbox_mode`, and the desktop app ignores custom profiles
  (openai/codex#28281). Prefer the classic keys.

## Manual follow-ups

- `codex` → sign in on first use.
