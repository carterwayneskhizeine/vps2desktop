# WebSSH (browser-based SSH client)

| | |
|---|---|
| id | `webssh` |
| group | apps |
| deps | `base-tools`, `uv` |
| channel | `uv-tool` (PyPI `webssh`, homepage = huashengdun/webssh) |

`huashengdun/webssh` — a Tornado + Paramiko + xterm.js web app that opens SSH
sessions in the browser over WebSockets. Installs the `wssh` executable; there is
no config file, no daemon package and no build step.

webssh has **no authentication of its own** — anyone who can load the page gets a
working SSH login form to type credentials into. Bind it to a single private
address, never `0.0.0.0`, and put a TLS terminator in front.

## Usage

Open the tunnel hostname in any browser, fill the form, and an xterm.js terminal
appears. To reach the machine that runs webssh itself, use a hostname that
**resolves on that machine** — see the resolution pitfall below.

| Field | Value for reaching the webssh host itself |
|---|---|
| Hostname | `127.0.0.1` (or `ubuntu` → `127.0.1.1` via `/etc/hosts`) |
| Port | `22` |
| Username | the target account |
| Password | that account's password |
| Private Key | leave empty for password auth |
| Passphrase / Totp | leave empty unless the key/second factor needs them |

The Hostname field is **not** resolved by your browser — webssh calls
`socket.getaddrinfo()` on the server. A name that works in your laptop's address
bar will not necessarily resolve there.

**Pick the bind address from where the TLS terminator lives.** A *containerized*
Cloudflare Tunnel resolves `host.docker.internal` to the Docker bridge gateway
(`172.17.0.1`), not to `127.0.0.1`, so a loopback-bound webssh is unreachable
and every request 502s. Bind the bridge gateway in that case; bind `127.0.0.1`
only when the terminator runs on the host (nginx, `tailscale serve`) or you
front it with an SSH tunnel. See Pitfalls.

## Guard (idempotency)

```bash
export PATH=/root/.local/bin:$PATH
command -v wssh && wssh --version
systemctl is-active --quiet wssh && ss -ltn | grep -qw 8127
```

All three passing means it is installed and serving — report `wssh --version` and skip.
The Guard only proves it is listening; confirm the *address* separately, since a
loopback bind in front of a containerized tunnel looks installed and healthy.


## Install

```bash
set -euo pipefail
source /etc/os-release
test "$ID" = ubuntu
test "$(dpkg --print-architecture)" = amd64
export PATH=/root/.local/bin:$PATH

# PyPI `webssh` IS the huashengdun project (home_page == the GitHub repo).
# uv keeps it in an isolated venv at /root/.local/share/uv/tools/webssh.
uv tool install webssh
```

`uv tool install` puts the venv under `/root/.local/share/uv/tools/webssh` and the
`wssh` shim on PATH at `/root/.local/bin/wssh`. Do **not** `pip install webssh`
into the system or conda Python — `python3` on these machines resolves to
`/root/anaconda3/bin/python3`, and webssh's pinned `paramiko`/`cryptography` wheels
would land in Anaconda's environment.

### systemd service

Determine the address the terminator will dial, then bind exactly that:

```bash
# Is the TLS terminator a container? If so it will NOT reach 127.0.0.1.
docker inspect cloudflare-tunnel --format \
  '{{.HostConfig.NetworkMode}} {{.HostConfig.ExtraHosts}}' 2>/dev/null
docker network inspect bridge --format '{{range .IPAM.Config}}{{.Gateway}}{{end}}'
```

For a containerized Cloudflare Tunnel the origin is `172.17.0.1`:

```bash
install -m 0644 /dev/stdin /etc/systemd/system/wssh.service <<'EOF'
[Unit]
Description=WebSSH — browser-based SSH client (huashengdun/webssh)
Documentation=https://github.com/huashengdun/webssh
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/root/.local/bin/wssh --address=172.17.0.1 --port=8127 --policy=warning
Restart=always
RestartSec=3
KillSignal=SIGTERM

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now wssh
```

Substitute `127.0.0.1` in `ExecStart` if the terminator runs on the host.

A single-address bind is deliberate: it keeps an unauthenticated SSH-credential
form off the public NIC. Before choosing it, check what else can reach the bridge
gateway — `docker network inspect bridge` showed no containers on the default
bridge, so the added exposure was nil on this machine.

The service runs as **root**. That matches this repository's root-managed model
(same reasoning as `docker-engine`), and webssh holds no privileges of its own —
it only brokers paramiko connections to whatever host the operator types in.

## Verify

`ORIGIN` must be the address the TLS terminator actually dials — `127.0.0.1` for a
host-side terminator, `172.17.0.1` for a containerized Cloudflare Tunnel.

```bash
export PATH=/root/.local/bin:$PATH
ORIGIN=172.17.0.1

wssh --version                                   # 1.6.3
systemctl is-active --quiet wssh && echo active
systemctl is-enabled --quiet wssh && echo enabled

# The bind address is the thing that actually breaks — assert it, don't eyeball it
ss -ltn | grep -qw 8127 && ! ss -ltn | grep -qw '0.0.0.0:8127'

curl -sS -o /dev/null -w '%{http_code}\n' http://$ORIGIN:8127/                    # 200
curl -sS -o /dev/null -w '%{http_code}\n' http://$ORIGIN:8127/static/js/xterm.min.js  # 200
curl -sS -o /dev/null -w '%{http_code}\n' http://$ORIGIN:8127/static/js/main.js       # 200

# WebSocket upgrade — 101 Switching Protocols proves the terminal channel works.
# curl then hangs holding the upgraded connection; that timeout is expected.
curl -sS -o /dev/null -m 5 -w '%{http_code}\n' http://$ORIGIN:8127/ws \
  -H 'Connection: Upgrade' -H 'Upgrade: websocket' \
  -H 'Sec-WebSocket-Version: 13' -H 'Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ=='   # 101

# End-to-end through the tunnel — proves origin bind, Host preservation and TLS
curl -sS -o /dev/null -m 20 -w '%{http_code}\n' https://<tunnel-hostname>/        # 200

# Nothing new in the tunnel log
docker logs --since 3m cloudflare-tunnel 2>&1 | grep -E 'ERR|WRN' | tail -5
```

Asset paths are **relative** (`static/...`, no leading slash) — `/xterm/css/xterm.css`
and `/static/main.js` both 404 and look like a broken install. Read the paths out of
the served page instead of guessing.

Record the version only after every command above passes.

## Known Pitfalls

- **The Hostname field is resolved by the *server*, not the browser.** webssh runs
  `getaddrinfo()` on its own host, so a name that resolves on your laptop fails here
  with `socket.gaierror: [Errno -2] Name or service not known`, surfaced as
  `Unable to connect to <host>:<port>`. **This happens before authentication**, so a
  correct password still gets rejected and the message misleadingly looks like a
  credential problem. Always test the name from the server:
  `getent hosts <name>` — for the webssh host itself use `127.0.0.1`, or the
  `/etc/hosts` alias `ubuntu` → `127.0.1.1`.
- **Tailscale DNS can rewrite the address you typed.** With MagicDNS enabled,
  `getent hosts 38.60.204.237` returned `100.119.149.52 … light-node-01.tail2398a5.ts.net`
  — the public IP resolves to the tailnet address, so a session intended for the
  public path can silently land on the Tailscale interface.
- **`uv tool install` inherits whichever Python it finds first — here Anaconda.**
  `readlink -f /root/.local/share/uv/tools/webssh/bin/python` →
  `/root/anaconda3/bin/python3.14`. site-packages are isolated, but the **stdlib is
  not**: tracebacks show `/root/anaconda3/lib/python3.14/socket.py` and
  `concurrent/futures/_base.py` alongside the venv's `paramiko`. Upgrading or
  removing Anaconda would then break webssh in a way that does not name the cause.
  To decouple, reinstall against the system interpreter:
  `uv tool install --python /usr/bin/python3 --force webssh` (drops to 3.12).
- **A loopback bind 502s behind a containerized Cloudflare Tunnel.** The tunnel runs
  in a container with `--add-host host.docker.internal:host-gateway`, so it dials the
  Docker bridge gateway `172.17.0.1` — never `127.0.0.1`. `check_origin` and the
  WebSocket path never even get exercised; the kernel refuses the connection:
  `Unable to reach the origin service ... dial tcp 172.17.0.1:8127: connect:
  connection refused`. Note the container may sit on `172.19.0.0/16` while
  `host-gateway` still resolves to `172.17.0.1` — the *default bridge* gateway, not
  the tunnel network's. Read the address out of the tunnel log, do not infer it.
  `docker logs cloudflare-tunnel | grep 'Unable to reach'` is the fastest diagnosis,
  and per-origin refusal counts tell you which routes share the fault.
- **The same trap hit a sibling route on the same machine**: a Vite dev server
  (`host: '127.0.0.1'`) behind `aios-light.goldie-rill.top` had logged 8 identical
  refusals and looked healthy only because nobody had requested it. When one
  tunnel route 502s, sweep the others for loopback-bound origins rather than
  assuming the app is uniquely broken.
- **A loopback bind is not a firewall.** On this machine `ufw` is absent and the
  iptables INPUT policy is ACCEPT, so binding `0.0.0.0` would publish webssh on the
  public IP. Bind the bridge gateway, not the wildcard.
- **No built-in auth.** webssh renders a login form for *other* machines' credentials,
  so any route to it is an open credential-phishing page. Consider a Cloudflare
  Access policy in front of the hostname.
- **`--fbidhttp` (default `True`) rejects plain HTTP from non-private source IPs** —
  `handler.py` `is_forbidden()` returns 403 unless `ip_address.is_private`. Python's
  `is_private` is False for Tailscale's `100.64.0.0/10` CGNAT range, so binding the
  Tailscale IP and browsing over plain http yields **403 "Public plain http request is
  forbidden."** Either tunnel it (TLS terminator ⇒ never plain http to the app) or pass
  `--fbidhttp=False` knowingly.
- **Behind a reverse proxy, the origin check compares `Origin` netloc to the `Host`
  header including port** (`check_origin()`, policy `same` by default, no fallback).
  Cloudflare Tunnel preserves `Host` by default, so `Origin: https://ssh.example.com`
  matches and it works. If the ingress rewrites `Host` — Cloudflare's `http2Origin:
  true`, or an nginx `proxy_set_header Host $host` mistake — the session-opening POST
  fails with **403 "Cross origin operation is not allowed."** Fix the header, don't
  reach for `--origin='*'` (allowed in debug mode only).
- **`--xheaders` defaults to `True`**, so tornado overwrites `request.protocol` from
  `X-Forwarded-Proto`; `is_forbidden()` then skips its plain-http branch entirely. That
  is what makes the tunnel path work, but it also means a proxy can spoof protocol —
  only expose it to a proxy you control.
- **`--policy` is the host-key policy for the *outbound* SSH session, not for webssh
  itself.** Default `warning` auto-adds unknown host keys and only logs. `--policy=reject`
  plus `--hostfile=/root/.ssh/known_hosts` pins host keys — the recommended setting when
  the browser sessions target machines whose host keys are already known.
- **`--maxconn` is 20 live SSH sessions *per client IP*** — a second browser tab past the
  limit gets 403 "Too many live connections." Behind a proxy every session looks like it
  comes from the same bridge gateway, so the limit is effectively global for tunneled
  traffic.
- **`wssh` is a uv tool**: upgrades are `uv tool upgrade webssh` (or `uv tool install
  --force webssh` to pull a new version). `pip list` will not show it, and
  `uv tool list` will. The venv is under `/root/`, so a service running as a
  non-root user cannot traverse it (`/root` is `drwx--x--x`) — running the unit as a
  dedicated unprivileged user requires installing outside `/root`.
- Verified live on light-node-01 on 2026-10-17: webssh 1.6.3 with tornado 6.5.10 and
  paramiko 5.0.0 under Python 3.14; page, assets, `101` WebSocket upgrade and an
  in-process paramiko handshake against the local `sshd` (`ssh-ed25519`,
  OpenSSH 9.6p1) all confirmed, then end-to-end 200 over
  `https://webssh.goldie-rill.top/` after rebinding to `172.17.0.1`.

