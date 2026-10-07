# Tailscale

| | |
|---|---|
| id | `tailscale` |
| group | base |
| deps | none |
| channel | apt (official stable repository, always latest) |

Private networking between devices in a tailnet. Installs the CLI and enables
the `tailscaled` service; account login and joining the tailnet are manual.

Official sources: [Linux installation](https://tailscale.com/kb/1031/install-linux)
and [stable packages](https://pkgs.tailscale.com/stable/).

Follow SKILL.md's identity check first. Run locally when already on the target
VPS; otherwise reuse the session's SSH connection. All commands below run as root.

## Guard (idempotency)

```bash
command -v tailscale >/dev/null 2>&1 &&
  dpkg-query -W -f='${Status}' tailscale 2>/dev/null | grep -qx 'install ok installed'
```

If present, run Verify. Skip only when Verify passes; otherwise run Install to
repair the package/service. An unauthenticated but healthy daemon is a valid
installation and must not trigger repeated reinstalls or automatic login.

## Install

The session's preflight `apt-get update` must have completed first. Adding this
vendor repository requires another index refresh before installing Tailscale.

```bash
set -euo pipefail
export DEBIAN_FRONTEND=noninteractive
apt-get install -y ca-certificates curl
. /etc/os-release
test "$ID" = ubuntu
test "$(uname -m)" = x86_64
test "${VERSION_ID%%.*}" -ge 24
test -n "${VERSION_CODENAME:-}"

TS_REPO_TMP=$(mktemp -d)
trap 'rm -rf "$TS_REPO_TMP"' EXIT
curl --fail --location --retry 3 \
  "https://pkgs.tailscale.com/stable/ubuntu/${VERSION_CODENAME}.noarmor.gpg" \
  --output "$TS_REPO_TMP/tailscale-archive-keyring.gpg"
curl --fail --location --retry 3 \
  "https://pkgs.tailscale.com/stable/ubuntu/${VERSION_CODENAME}.tailscale-keyring.list" \
  --output "$TS_REPO_TMP/tailscale.list"
install -d -m 0755 /usr/share/keyrings /etc/apt/sources.list.d
install -m 0644 "$TS_REPO_TMP/tailscale-archive-keyring.gpg" /usr/share/keyrings/tailscale-archive-keyring.gpg
install -m 0644 "$TS_REPO_TMP/tailscale.list" /etc/apt/sources.list.d/tailscale.list
apt-get update
apt-get install -y tailscale
systemctl enable --now tailscaled
```

## Verify

```bash
set -euo pipefail
test "$(dpkg-query -W -f='${Status}' tailscale)" = 'install ok installed'
/usr/bin/tailscale version
systemctl is-enabled tailscaled
systemctl is-active tailscaled
python3 - <<'PY'
import json
import subprocess
status = json.loads(subprocess.check_output(['/usr/bin/tailscale', 'status', '--json'], text=True))
assert status.get('BackendState'), 'Tailscale daemon did not report its state'
print('Tailscale backend state:', status['BackendState'])
PY
```

Capture the first line of `tailscale version` as the installed version.
`NeedsLogin` is expected before the manual follow-up; Verify confirms the
installation and daemon, not membership in a tailnet. Do not print the complete
status JSON: it can include account details, device addresses and login URLs.

## Manual follow-ups

- The user runs `sudo tailscale up` and opens the displayed authentication URL
  to sign in and authorize the device. Do not automate OAuth or request an auth key.
- After authorization, the user can run `tailscale status` and `tailscale ip`
  to confirm membership and view its tailnet addresses.

## Desktop integration (optional)

The installed Linux CLI includes `tailscale systray`. First ensure the Xfce
status tray using the procedure in `xfce-desktop.md`. Use the actual desktop
session's DISPLAY and D-Bus address for every graphical command; a shell in the
same VPS is not necessarily part of that graphical session.

Before enabling startup, launch the bundled tray detached in the desktop
session and verify its StatusNotifierItem and menu registration:

```python
import os
import subprocess
from pathlib import Path

pids = subprocess.check_output(['pgrep', '-u', str(os.getuid()), '-x', 'xfce4-session'],
                               text=True).split()
assert len(pids) == 1, 'Exactly one target desktop session is required'
values = Path(f'/proc/{pids[0]}/environ').read_bytes().split(b'\0')
session = dict(v.decode().split('=', 1) for v in values if b'=' in v)
env = os.environ.copy()
for key in ('DISPLAY', 'DBUS_SESSION_BUS_ADDRESS', 'XDG_RUNTIME_DIR', 'XAUTHORITY'):
    if key in session:
        env[key] = session[key]
assert env.get('DISPLAY') and env.get('DBUS_SESSION_BUS_ADDRESS')
subprocess.Popen(['/usr/bin/tailscale', 'systray', '--theme=light:nobg'],
                 env=env, stdin=subprocess.DEVNULL, stdout=subprocess.DEVNULL,
                 stderr=subprocess.DEVNULL, start_new_session=True)
print('Started bundled Tailscale tray in the existing desktop session.')
```

This is a compatibility check, not a universal guarantee: upstream currently
lists Xfce as unsupported and advises against running systray as root. The
current upstream code warns about root rather than unconditionally exiting.
Only persist it on a root Xfce desktop after verifying that its icon and menu
work in that exact session. Do not change accounts, Tailscale operator settings
or the user's existing login to work around a GUI limitation.

For a graphical management entry point, Tailscale provides a local web UI at
`http://100.100.100.100/` on current clients. Check its HTTP response before
creating a desktop/menu launcher using the root-safe default browser. Changing
settings in this UI can require the user's manual browser authentication.

After the compatibility check passes, persist the tray using the official
freedesktop autostart command, run with the same session environment:

```bash
/usr/bin/tailscale configure systray --enable-startup=freedesktop
```

The generated `/root/.config/autostart/tailscale-systray.desktop` can use
`Exec=/usr/bin/tailscale systray --theme=light:nobg` for a dark panel. Keep its
other fields intact. This autostart is per desktop login; `tailscaled` itself
remains a separate system service.

For the desktop and application-menu shortcut, create
`/root/.local/share/applications/vps2desktop-tailscale.desktop`:

```ini
[Desktop Entry]
Version=1.0
Type=Application
Name=Tailscale
Comment=Open the Tailscale device management interface
Exec=exo-open --launch WebBrowser http://100.100.100.100/
Icon=network-vpn
Terminal=false
Categories=Network;
StartupNotify=true
```

Copy it to the desktop directory reported by `xdg-user-dir DESKTOP` as
`Tailscale.desktop`, make that copy executable and mark it trusted with
`gio set <desktop-file> metadata::trusted true` in the live desktop session.
Validate both launchers with `desktop-file-validate`. Use the root-safe browser
wrappers provided by `chrome.md`; a direct root launch of unwrapped Chrome fails.

Verify that the watcher reports both an active Tailscale StatusNotifierItem
with an icon and a working `com.canonical.dbusmenu.GetLayout`, and that the
local web UI responds successfully. Do not dump private status or menu details
into logs. If compatibility verification fails, do not enable tray autostart.

References: [Linux systray](https://tailscale.com/docs/features/client/linux-systray),
[current systray source](https://github.com/tailscale/tailscale/blob/main/client/systray/systray.go)
and [device web interface](https://tailscale.com/docs/features/client/device-web-interface).

## Known Pitfalls

- The package can be correctly installed while the backend reports `NeedsLogin`.
  Starting `tailscaled` alone does not join a tailnet.
- Kernel networking requires `/dev/net/tun`. If missing, investigate the VPS's
  TUN support; official guidance uses `modprobe tun`. Do not silently change to
  userspace networking, whose behavior differs from a TUN interface.
- Use the repository for the actual Ubuntu codename. If upstream does not yet
  publish that codename, stop on the download failure; do not substitute another
  distribution or the unstable channel.
- Exit-node, subnet-route, DNS and Tailscale SSH settings require a separate
  request; ordinary installation does not configure these features.
