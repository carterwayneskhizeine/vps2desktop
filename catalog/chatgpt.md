# ChatGPT desktop

| | |
|---|---|
| id | `chatgpt` |
| group | apps |
| deps | `xfce-desktop` |
| channel | vendor-deb (explicitly selected local amd64 package) |

Install a user-selected ChatGPT DEB and configure launchers for the root Xfce
desktop. Set `CHATGPT_DEB` to the absolute path of the package the user requested.
An explicit local-package request takes precedence over the protocol's default
latest-version policy; do not substitute another download or pin its version in
the Catalog. This component requires that package to be supplied and is not in
the Full preset.

The inspected vendor package includes a signed APT repository configuration in
its maintainer scripts. Let the package manage that configuration. Account login
remains manual.

## Guard (idempotency)

Run Verify as well before recording an existing installation as skipped.

```bash
set -euo pipefail
test "$(dpkg-query -W -f='${db:Status-Status}' chatgpt 2>/dev/null)" = installed
if [ -n "${CHATGPT_DEB:-}" ]; then
  test "$(dpkg-query -W -f='${Version}' chatgpt)" = "$(dpkg-deb --field "$CHATGPT_DEB" Version)"
fi
test -x /usr/lib/chatgpt/ChatGPT
test -x /usr/local/bin/chatgpt
grep -q -- '--no-sandbox' /usr/local/bin/chatgpt
grep -q '^Exec=/usr/local/bin/chatgpt %U$' /root/.local/share/applications/chatgpt.desktop
test -x /root/Desktop/ChatGPT.desktop
echo present
```

## Install

Run after the protocol's machine identity checks and APT index refresh.

```bash
set -euo pipefail
: "${CHATGPT_DEB:?Set CHATGPT_DEB to the user-selected absolute DEB path}"
case "$CHATGPT_DEB" in /*) ;; *) exit 1;; esac
test -r "$CHATGPT_DEB"
test "$(id -u)" = 0
test "$(dpkg --print-architecture)" = amd64
test "$(dpkg-deb --field "$CHATGPT_DEB" Package)" = chatgpt
test "$(dpkg-deb --field "$CHATGPT_DEB" Architecture)" = amd64

DEBIAN_FRONTEND=noninteractive apt-get install -y "$CHATGPT_DEB"

install -d /usr/local/bin /root/.local/share/applications /root/Desktop
cat > /usr/local/bin/chatgpt <<'EOF'
#!/bin/sh
if [ "$(id -u)" -eq 0 ]; then
  exec /usr/bin/chatgpt --no-sandbox "$@"
fi
exec /usr/bin/chatgpt "$@"
EOF
chmod 755 /usr/local/bin/chatgpt

# Use a per-user override rather than modifying the package-owned launcher.
cp /usr/share/applications/chatgpt.desktop /root/.local/share/applications/chatgpt.desktop
sed -i 's#^Exec=chatgpt %U$#Exec=/usr/local/bin/chatgpt %U#' /root/.local/share/applications/chatgpt.desktop
grep -q '^Exec=/usr/local/bin/chatgpt %U$' /root/.local/share/applications/chatgpt.desktop
install -m 755 /root/.local/share/applications/chatgpt.desktop /root/Desktop/ChatGPT.desktop
update-desktop-database /root/.local/share/applications
```

Trust the desktop launcher using the existing desktop session's environment.
If there is no live session, defer this step until the first desktop login.

```python
import os
import subprocess
from pathlib import Path

pids = subprocess.check_output(
    ['pgrep', '-u', str(os.getuid()), '-x', 'xfce4-session'], text=True).split()
assert len(pids) == 1, 'Exactly one target Xfce session is required'
values = Path(f'/proc/{pids[0]}/environ').read_bytes().split(b'\0')
session = dict(v.decode().split('=', 1) for v in values if b'=' in v)
env = os.environ.copy()
for key in ('DISPLAY', 'DBUS_SESSION_BUS_ADDRESS', 'XDG_RUNTIME_DIR', 'XAUTHORITY'):
    if key in session:
        env[key] = session[key]
assert env.get('DISPLAY') and env.get('DBUS_SESSION_BUS_ADDRESS')
subprocess.run(['gio', 'set', '/root/Desktop/ChatGPT.desktop',
                'metadata::trusted', 'true'], env=env, check=True)
print('ChatGPT desktop launcher is trusted.')
```

## Verify

```bash
set -euo pipefail
test "$(dpkg-query -W -f='${db:Status-Status}' chatgpt)" = installed
if [ -n "${CHATGPT_DEB:-}" ]; then
  test "$(dpkg-query -W -f='${Version}' chatgpt)" = "$(dpkg-deb --field "$CHATGPT_DEB" Version)"
fi
test -x /usr/lib/chatgpt/ChatGPT
test -x /usr/local/bin/chatgpt
sh -n /usr/local/bin/chatgpt
grep -q -- '--no-sandbox' /usr/local/bin/chatgpt
grep -q '^Exec=/usr/local/bin/chatgpt %U$' /root/.local/share/applications/chatgpt.desktop
test -x /root/Desktop/ChatGPT.desktop
cmp /root/.local/share/applications/chatgpt.desktop /root/Desktop/ChatGPT.desktop
desktop-file-validate /root/.local/share/applications/chatgpt.desktop
if ldd /usr/lib/chatgpt/ChatGPT | grep -q 'not found'; then
  echo 'ChatGPT has missing shared libraries' >&2
  exit 1
fi
dpkg-query -W -f='${Package} ${Version}\n' chatgpt
```

When a desktop session is active, also verify a GUI launch before recording the
installation. Reuse the session environment above and launch
`/usr/local/bin/chatgpt` with stdin/stdout/stderr redirected and a detached
process session. Use `xprop -root _NET_CLIENT_LIST` and each window's
`_NET_WM_PID` to confirm a visible window belongs to
`/usr/lib/chatgpt/ChatGPT`. Do not enter credentials or automate login. With no
desktop session, report the GUI check as a manual follow-up.

## Known Pitfalls

- The packaged `/usr/bin/chatgpt` launcher forwards arguments to the bundled
  executable without adding root-session flags. Root launches need
  `--no-sandbox`; preserve the wrapper and the per-user desktop override.
- Package filenames do not establish the package name, architecture or version.
  Inspect the DEB metadata before installing and record the verified `dpkg-query`
  version rather than the bundled Chromium runtime's `version` file.
- GUI checks need the live session's DISPLAY, X authority and D-Bus address.
  A headless shell by itself cannot establish that a desktop window opens.
- The package has TPM-library dependencies; use APT to resolve them rather than
  installing the DEB with `dpkg -i` alone.
- **The app's data directory is not named after the package.** The Chromium
  profile lives in `~/.config/Codex` (`UserDataDirectoryName=Codex` in
  `resources/owl-app.ini`), while the shared account session lives in
  `~/.codex/auth.json`. Looking for `~/.config/chatgpt` finds nothing and can
  be mistaken for "the app was never started".
- **A stale `id_token` is not a logged-out app.** The id token in
  `~/.codex/auth.json` expires within hours; the session is carried by the
  refresh token. Judge the sign-in by `codex login status`, not by the id
  token's expiry.

## Manual follow-ups

- Sign in to ChatGPT in the application.

### Verifying the sign-in afterwards

The login itself is always manual, but its result can be checked read-only.
Never print token values, and never automate a login flow.

```bash
codex login status
```

`Logged in using ChatGPT` means the account session is live; an API-key
message means the app is signed in with a key instead. The desktop app and the
Codex CLI share one session, stored in `~/.codex/auth.json` (`auth_mode:
chatgpt`, plus id/access/refresh tokens and an account id). Inspect the key
names and the `auth_mode` only — never the token values.

The desktop app is an Owl/Electron build whose Chromium profile is
`~/.config/Codex`, not `~/.config/chatgpt`: `resources/owl-app.ini` sets
`UserDataDirectoryName=Codex`. That directory is separate from `~/.codex`,
which is the Codex CLI's own home, so the two coexist without conflict.
