# Xfce desktop

| | |
|---|---|
| id | `xfce-desktop` |
| group | desktop |
| deps | none |
| channel | apt |

Xfce 4 session, panel, terminal — plus dbus-x11 and Noto CJK fonts (Chinese/Japanese/Korean glyphs render correctly out of the box).

## Guard (idempotency)

```bash
dpkg -s xfce4 xfce4-terminal dbus-x11 fonts-noto-cjk >/dev/null 2>&1 && echo present
```

## Install

```bash
export DEBIAN_FRONTEND=noninteractive
apt-get install -y xfce4 xfce4-terminal dbus-x11 fonts-noto-cjk
```

### Ensure the status tray in an existing desktop session

The `systray` plugin is included in modern xfce4-panel; a separate
xfce4-statusnotifier-plugin package is deprecated. It displays both traditional
tray icons and StatusNotifierItems. The `notification-plugin` handles desktop
notifications and is not a substitute for a tray.

Run this after the first Xfce login, on the target machine as the desktop user
(root in this repository). It uses the actual session's D-Bus address and
preserves the existing panel items. If no session exists, defer this step until
the first login; do not write XML while xfconfd owns the live configuration.

```python
import os
import re
import subprocess
from pathlib import Path

pids = subprocess.run(['pgrep', '-u', str(os.getuid()), '-x', 'xfce4-session'],
                      capture_output=True, text=True).stdout.split()
assert len(pids) == 1, 'Exactly one target Xfce session is required'
values = Path(f'/proc/{pids[0]}/environ').read_bytes().split(b'\0')
session = dict(v.decode().split('=', 1) for v in values if b'=' in v)
env = os.environ.copy()
for key in ('DISPLAY', 'DBUS_SESSION_BUS_ADDRESS', 'XDG_RUNTIME_DIR', 'XAUTHORITY'):
    if key in session:
        env[key] = session[key]
assert env.get('DISPLAY') and env.get('DBUS_SESSION_BUS_ADDRESS')

def query(*args):
    return subprocess.check_output(['xfconf-query', '-c', 'xfce4-panel', *args],
                                   env=env, text=True)

properties = query('-lv')
plugins = {int(i): name.strip() for i, name in
           re.findall(r'^/plugins/plugin-(\d+)\s+(\S+)\s*$', properties, re.M)}
ids = [int(v) for v in query('-p', '/panels/panel-1/plugin-ids').splitlines()
       if v.strip().isdigit()]
assert ids, 'Top panel not found; do not overwrite a custom layout'
tray_id = next((i for i in ids if plugins.get(i) == 'systray'), None)
if tray_id is None:
    tray_id = max(plugins, default=0) + 1
    query('-p', f'/plugins/plugin-{tray_id}', '-n', '-t', 'string', '-s', 'systray')
    position = next((n for n, i in enumerate(ids)
                     if plugins.get(i) in ('pulseaudio', 'clock', 'actions')), len(ids))
    ids.insert(position, tray_id)
    args = ['-p', '/panels/panel-1/plugin-ids', '-a']
    for value in ids:
        args.extend(['-t', 'int', '-s', str(value)])
    query(*args)
print('Top panel status tray is configured; existing panel items preserved.')
```

Verify the session bus's `org.kde.StatusNotifierWatcher` and its
`RegisteredStatusNotifierItems` property. If the configuration has changed but
the live panel has not loaded the tray, restart only `xfce4-panel` in that same
session (`xfce4-panel --restart`, detached with stdin/stdout/stderr redirected).
This refreshes the panel without logging out or closing desktop applications.
Back up the panel configuration to Local State before restarting. If the panel
lost its session-bus connection, terminate only the confirmed panel PID and
start a replacement using the same session environment; do not kill the session.

Official references: [Notification Area](https://docs.xfce.org/xfce/xfce4-panel/systray)
and [Status Notifier integration](https://docs.xfce.org/panel-plugins/xfce4-statusnotifier-plugin/start).

## Verify

```bash
dpkg -s xfce4 | grep -q 'Status: install ok installed' && xfce4-terminal --version 2>/dev/null | head -1
```

## Known Pitfalls

- This only installs the desktop; remote access comes from the `xrdp` component.
- ~400+ packages with dependencies on a minimal image — expected, not an error.
- Input methods can keep working even when their icons are absent: check that
  `systray` is present in the live top panel and the item is not hidden in its
  properties. With a running desktop, the package Guard alone does not verify
  tray configuration; run the status-tray step when repairing missing icons.
