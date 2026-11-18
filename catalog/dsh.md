# DeepSeek Harness (dsh)

| | |
|---|---|
| id | `dsh` |
| group | ai-cli |
| deps | `node-nvm` |
| channel | npm (@deepseek-ai/dsh) |

DeepSeek's open agent harness as a global npm package under the nvm-managed
Node — coding agent runtime plus a bundled web UI (`dsh web`). Sessions,
profiles and storages live under `~/.dsh/`.

## Guard (idempotency)

```bash
ls /root/.nvm/versions/node/*/bin/dsh >/dev/null 2>&1 && echo present
```

## Install

```bash
for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done
npm install -g @deepseek-ai/dsh
```

## Verify

```bash
for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done
dsh --version
DSH_ROOT="$(npm root -g)/@deepseek-ai/dsh"
node -e "require('$DSH_ROOT/node_modules/node-pty')" && echo node-pty loads
```

`dsh --version` proves the CLI; the `node -e` line proves the native pty
module the terminal/subprocess path depends on actually loads.

## Running the web UI

`dsh web` is a foreground process (Ctrl-C stops it); it does not daemonize
and ships no service unit. Run it in tmux or write your own unit. It prints
a startup URL containing a one-shot token; the token is rotated on every
restart, so re-read the pane after restarting.

```bash
dsh web --no-open --host 172.17.0.1 --port 3080 \
  --public-url https://<public-hostname>/ --trusted-host <public-hostname>
```

- `--no-open`: skip launching a browser (headless server).
- `--host`: bind exactly one private address. `127.0.0.1` for a host-local
  proxy; `172.17.0.1` (the Docker bridge gateway) when a *containerized*
  Cloudflare Tunnel must reach it — the container cannot see loopback, and a
  `127.0.0.1` bind 502s behind such a tunnel. Never `0.0.0.0`.
- `--public-url` / `--trusted-host`: must match the public HTTPS hostname in
  front of the listener, or the host-trust check rejects requests.

## Known Pitfalls

- **npm ≥ 12 blocks install scripts by default** (`node-pty`, `koffi`,
  `protobufjs`, `@google/genai`, `@deepseek-ai/dsh-subprocess-local` all warn).
  The install still works: `node-pty` ships prebuilds for every platform
  including linux-x64, so the CLI runs without them. Only rerun with
  `--allow-scripts=node-pty,koffi,protobufjs,@google/genai,@deepseek-ai/dsh-subprocess-local`
  if Verify shows a native module failing to load.
- **The `latest` dist-tag lags `next`** (measured: `latest=0.2.2-rc.1` while
  `next=0.2.2-rc.4`). Check `npm view @deepseek-ai/dsh dist-tags` and install
  `@deepseek-ai/dsh@next` (or an explicit version) when you want the newest
  release; a plain `npm install -g @deepseek-ai/dsh` reinstalls `latest`.
- **Upgrades take effect only after a restart.** `npm install -g` replaces
  the on-disk package while a running `dsh web` keeps the old code in memory.
  Restarting interrupts any in-flight agent turn — check the process has no
  children first (`ps --ppid <pid>`) — and rotates the startup token.
- **No TLS of its own.** The listener is plain HTTP and the session cookie is
  not `Secure`; dsh neither issues nor renews certificates. Terminate TLS in
  front (tunnel / reverse proxy) or pass `--tls-cert` / `--tls-key` with a
  certificate covering the hostname actually used to reach it.

## Manual follow-ups

- Open the startup URL (token from the `dsh web` output), then paste a
  DeepSeek API key in Settings → Models. It is stored in
  `~/.dsh/.credentials.yaml` (0600) — never automate or commit it.
