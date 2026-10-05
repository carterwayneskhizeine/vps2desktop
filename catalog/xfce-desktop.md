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

## Verify

```bash
dpkg -s xfce4 | grep -q 'Status: install ok installed' && xfce4-terminal --version 2>/dev/null | head -1
```

## Known Pitfalls

- This only installs the desktop; remote access comes from the `xrdp` component.
- ~400+ packages with dependencies on a minimal image — expected, not an error.
