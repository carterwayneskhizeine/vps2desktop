# Chrome MCP Server (mcp-chrome)

| | |
|---|---|
| id | `mcp-chrome` |
| group | `ai-cli` |
| deps | `base-tools`, `node-nvm` |
| channel | npm (bridge) + vendor-download (Chrome for Testing, extension zip) |

Drives **your own** Chrome as an MCP server: an unpacked extension talks to a Node
native-messaging bridge, and the bridge exposes a streamable-HTTP MCP endpoint on
`127.0.0.1:12306/mcp` (27 tools: navigate, screenshot, click/fill, read page, network
capture, history, bookmarks, console, GIF recorder). Because the profile is a real,
persistent Chrome profile, cookies and logins survive restarts — unlike a throwaway
Playwright context.

```
MCP client ──HTTP──> bridge 127.0.0.1:12306
                        ▲  stdio (native messaging)
                        │
                    extension ── debugger/scripting ──> Chrome (headless, own profile)
```

The stack is three pieces that fail independently: the npm bridge, the extension, and
the Chrome binary. Branded `google-chrome` cannot host the extension (see Known
Pitfalls), so this Component ships **Chrome for Testing** — a vendor build that still
honours `--load-extension` — and runs the whole thing under an unprivileged account so
Chrome keeps its own sandbox instead of needing `--no-sandbox` on a root-managed VPS.

## Guard (idempotency)

```bash
test -x /opt/mcp-chrome/chrome/chrome &&
test -f /opt/mcp-chrome/extension/manifest.json &&
id -u mcpchrome >/dev/null 2>&1 &&
systemctl is-enabled --quiet mcp-chrome.service &&
curl -sf --max-time 5 http://127.0.0.1:12306/ping >/dev/null && echo present
```

## Install

### 1. The unprivileged run account

The browser, the extension's native host and its HTTP server all run as this account.
Reuse an existing browser run account if the machine already has one; this Component
creates `mcpchrome` when there is none.

```bash
id -u mcpchrome >/dev/null 2>&1 ||
  useradd -m -s /bin/bash -c "Unprivileged run account for mcp-chrome" mcpchrome
passwd -l mcpchrome >/dev/null 2>&1 || true

# The native host is node from the nvm tree under /root: the run account must be able
# to traverse /root (711, still not listable) and reach the Node binaries (755).
chmod 711 /root
chmod 755 /root/.nvm /root/.nvm/versions
```

Check both directions: `sudo -u mcpchrome /root/.nvm/versions/node/*/bin/node -v` must
work, `sudo -u mcpchrome ls /root` must fail.

### 2. The bridge and its native-messaging manifest

```bash
export PATH="$(ls -1d /root/.nvm/versions/node/*/bin | sort -V | tail -n1):$PATH"

# npm >= 12 blocks install scripts by default. The bridge's postinstall registers the
# host, and better-sqlite3 (history/bookmarks) needs its native build — allow both.
npm install -g --allow-scripts=mcp-chrome-bridge,better-sqlite3 mcp-chrome-bridge

# System-level registration writes /etc/opt/chrome/native-messaging-hosts/.
mcp-chrome-bridge register --system
```

Chrome for Testing does **not** read `/etc/opt/chrome`: it uses its own brand paths, so
copy the manifest there too (system-wide, so it covers any run account):

```bash
M=/etc/opt/chrome/native-messaging-hosts/com.chromemcp.nativehost.json
install -d -m 0755 /etc/opt/chrome_for_testing/native-messaging-hosts
install -m 0644 "$M" /etc/opt/chrome_for_testing/native-messaging-hosts/
```

### 3. The extension

```bash
install -d -m 0755 /opt/mcp-chrome/extension
curl --fail --location --retry 3 -o /tmp/chrome-mcp-server.zip \
  https://github.com/hangwin/mcp-chrome/raw/master/releases/chrome-extension/latest/chrome-mcp-server-lastest.zip
unzip -q -o /tmp/chrome-mcp-server.zip -d /tmp/chrome-mcp-unzip
src="$(dirname "$(find /tmp/chrome-mcp-unzip -name manifest.json | head -n1)")"
cp -a "${src}/." /opt/mcp-chrome/extension/
chmod -R a+rX /opt/mcp-chrome/extension
rm -rf /tmp/chrome-mcp-unzip /tmp/chrome-mcp-server.zip
```

The zip's manifest carries a `key`, which pins the extension id
(`hbdgbgagpkpjffpklnamcljpakneikee` for the current release) — that is the id the native
host manifest's `allowed_origins` lists, so the two must be installed as a pair.

### 4. Chrome for Testing

```bash
curl --fail --location --retry 3 -o /tmp/cft.json \
  https://googlechromelabs.github.io/chrome-for-testing/last-known-good-versions-with-downloads.json
url="$(python3 - <<'PY'
import json
ch = json.load(open("/tmp/cft.json"))["channels"]["Stable"]
print(next(d["url"] for d in ch["downloads"]["chrome"] if d["platform"] == "linux64"))
PY
)"
curl --fail --location --retry 3 -o /tmp/cft.zip "$url"
rm -rf /opt/mcp-chrome/chrome
unzip -q /tmp/cft.zip -d /opt/mcp-chrome
mv /opt/mcp-chrome/chrome-linux64 /opt/mcp-chrome/chrome
chmod -R a+rX /opt/mcp-chrome/chrome
rm -f /tmp/cft.zip /tmp/cft.json

# Ubuntu 23.10+ blocks unprivileged user namespaces with AppArmor, which leaves Chrome
# with "No usable sandbox!". Give this one binary a profile that grants userns, and keep
# the SUID helper setuid as the fallback path.
cat > /etc/apparmor.d/mcp-chrome <<'EOF'
abi <abi/5.0>,
include <tunables/global>

profile mcp-chrome /opt/mcp-chrome/chrome/chrome flags=(unconfined) {
  userns,
  @{exec_path} mr,

  include if exists <local/mcp-chrome>
}
EOF
apparmor_parser -r /etc/apparmor.d/mcp-chrome
chown root:root /opt/mcp-chrome/chrome/chrome_sandbox
chmod 4755 /opt/mcp-chrome/chrome/chrome_sandbox
```

### 5. Launcher and service

```bash
install -d -m 0755 /opt/mcp-chrome
cat > /opt/mcp-chrome/start-chrome.sh <<'EOF'
#!/usr/bin/env bash
# Chrome for Testing hosting the mcp-chrome extension. Runs as the unprivileged run
# account so Chrome keeps its own sandbox; /etc/apparmor.d/mcp-chrome grants the userns
# the sandbox needs on Ubuntu 23.10+.
set -euo pipefail

CHROME=/opt/mcp-chrome/chrome/chrome
EXT=/opt/mcp-chrome/extension
PROFILE=/var/lib/mcp-chrome/profile
PORT=9224
RUN_USER=mcpchrome

mkdir -p "$PROFILE"
chown -R "${RUN_USER}:${RUN_USER}" /var/lib/mcp-chrome

exec /usr/bin/setpriv --reuid="${RUN_USER}" --regid="${RUN_USER}" --init-groups \
  env HOME="/home/${RUN_USER}" \
  "$CHROME" \
  --headless=new \
  --remote-debugging-port=$PORT \
  --remote-allow-origins='*' \
  --user-data-dir="$PROFILE" \
  --load-extension="$EXT" \
  --no-first-run \
  --no-default-browser-check \
  --disable-dev-shm-usage \
  --window-size=1400,1000 \
  about:blank
EOF
chmod 0755 /opt/mcp-chrome/start-chrome.sh

# The extension writes its native-host logs and crash dirs under the run account's HOME;
# `install -d -o` only chowns the leaf, so a root-owned ~/.config makes crashpad abort.
install -d -o mcpchrome -g mcpchrome -m 0755 /home/mcpchrome/.config/google-chrome-for-testing/NativeMessagingHosts
chown -R mcpchrome:mcpchrome /home/mcpchrome/.config
install -o mcpchrome -g mcpchrome -m 0644 \
  /etc/opt/chrome/native-messaging-hosts/com.chromemcp.nativehost.json \
  /home/mcpchrome/.config/google-chrome-for-testing/NativeMessagingHosts/

install -d -o mcpchrome -g mcpchrome -m 0700 /var/lib/mcp-chrome

cat > /etc/systemd/system/mcp-chrome.service <<'EOF'
[Unit]
Description=Chrome for Testing + mcp-chrome extension (headless, CDP 9224)
After=network.target

[Service]
Type=simple
ExecStart=/opt/mcp-chrome/start-chrome.sh
Restart=always
RestartSec=5
KillMode=mixed
User=root
Environment=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now mcp-chrome.service
```

The extension auto-connects on Chrome start (`ensureNativeConnected("sw_startup")`), so
the HTTP endpoint appears within a few seconds without touching the popup.

## Verify

```bash
systemctl is-active mcp-chrome.service
curl -sf --max-time 5 http://127.0.0.1:12306/ping   # {"status":"ok","message":"pong"}
```

`/ping` only answers once the whole chain works — extension connected, native host alive,
HTTP server bound — so it is the real data-path check, not a process check. Then prove
the browser is sandboxed and not root:

```bash
ps -eo user,args | grep "[m]cp-chrome/chrome/chrome --headless" | cut -c1-80
# expect a non-root user and NO --no-sandbox in the command line

ps -eo args | grep "[m]cp-chrome/chrome/chrome --headless" | grep -c -- --no-sandbox   # 0
```

And a tool call end to end (one client at a time — see Known Pitfalls):

```bash
SID="$(curl -s -D - -o /dev/null -X POST http://127.0.0.1:12306/mcp \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"verify","version":"1"}}}' \
  | sed -n 's/^mcp-session-id: //p' | tr -d '\r')"
curl -s -X POST http://127.0.0.1:12306/mcp -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' -H "mcp-session-id: ${SID}" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"get_windows_and_tabs","arguments":{}}}'
# SSE frame with windowCount/tabCount, isError: false
systemctl restart mcp-chrome.service   # frees the transport for the next client
```

Client configuration for an MCP client that speaks streamable HTTP:

```json
{"mcpServers": {"streamable-mcp-server": {"type": "streamable-http", "url": "http://127.0.0.1:12306/mcp"}}}
```

## Known Pitfalls

- **Branded `google-chrome` silently ignores `--load-extension`.** Measured on
  google-chrome 157: the process starts, the extension never appears in
  `http://127.0.0.1:<port>/json`, and opening its `chrome-extension://<id>/…` URL returns
  `ERR_BLOCKED_BY_CLIENT`. Google removed sideloading through that switch for the branded
  builds; Chromium and **Chrome for Testing** still honour it. Do not "fix" this by
  switching to the desktop `chrome` component's binary.
- **Chrome for Testing uses its own native-messaging brand paths.** A manifest in
  `/etc/opt/chrome/native-messaging-hosts/` (what `mcp-chrome-bridge register --system`
  writes) is invisible to it: `connectNative` fails with *"Specified native messaging
  host not found."* and the popup shows `Service Not Started`. Install the manifest under
  `/etc/opt/chrome_for_testing/native-messaging-hosts/` and/or
  `~/.config/google-chrome-for-testing/NativeMessagingHosts/` as well.
- **"No usable sandbox!" on Ubuntu 23.10+.** `apparmor_restrict_unprivileged_userns=1`
  blocks the namespace sandbox for binaries without an AppArmor profile, and the SUID
  helper in the CfT zip is not setuid out of the box — so a non-root launch aborts with
  `FATAL:zygote_host_impl_linux.cc: No usable sandbox!`. Two changes, both in the install
  block: an AppArmor profile for `/opt/mcp-chrome/chrome/chrome` with `userns,`, and
  `chmod 4755 chrome_sandbox`. Verify with the `ps` checks above rather than assuming.
- **A root-owned `~/.config` kills Chrome with a confusing crash.** `install -d -o` chowns
  only the leaf directory; the intermediate `~/.config` then stays `root:root`, crashpad
  cannot create its database and Chrome dies with `chrome_crashpad_handler: --database is
  required` plus `SIGTRAP` (exit 133) before any window. `chown -R` the run account's
  `~/.config` after creating the native-messaging directory.
- **npm ≥ 12 blocks the install scripts the bridge needs.** A plain `npm install -g`
  leaves `mcp-chrome-bridge` without its postinstall (no host registration) and
  `better-sqlite3` without its native binding (history/bookmarks tools fail). Install with
  `--allow-scripts=mcp-chrome-bridge,better-sqlite3`.
- **The MCP endpoint holds exactly one client transport.** A second client's `initialize`
  returns `500 Already connected to a transport. Call close() before connecting to a new
  transport…`, and `DELETE /mcp` is answered with `400` — there is no clean in-band close.
  Free it by clicking **Disconnect** in the extension popup (then **Connect**), or by
  `systemctl restart mcp-chrome.service`. Plan for one MCP client per machine, or restart
  the service between clients.
- **Disconnect also disables auto-connect.** The popup's Disconnect writes
  `nativeAutoConnectEnabled=false` to `chrome.storage.local`, so after a service restart
  the endpoint stays down until someone clicks Connect. Clicking Connect sets it back to
  `true`. Check the flag before blaming the service: read
  `(await chrome.storage.local.get('nativeAutoConnectEnabled'))` from the extension's
  service worker, or simply make sure the last thing done in the popup was Connect.
- **The native host inherits the run account, so it must reach Node through `/root`.**
  The host is `run_host.sh` → `node …/dist/index.js` under `/root/.nvm`. With `/root` at
  mode 700 the host dies instantly and leaves an empty log dir; there is no error in the
  Chrome log. `/root` 711 plus `/root/.nvm` and `/root/.nvm/versions` 755 is the
  requirement (the same arrangement `browser-mcp` uses).
- **Headless Chrome does not beat Cloudflare.** A managed challenge stays on
  *"Performing security verification"* forever in this configuration (measured against a
  Cloudflare-protected storefront); the extension adds no evasion. For WAF-protected sites
  drive a **headful** Chrome against the RDP display instead of this service.
- **Loopback only, and no authentication.** `12306` binds `127.0.0.1` and every tool runs
  with the browser's cookies. Never publish the port through a tunnel or reverse proxy
  without an auth layer in front.
