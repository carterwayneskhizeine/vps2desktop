# Snipaste

| | |
|---|---|
| id | `snipaste` |
| group | apps |
| deps | none (pair with `xfce-desktop` for actual use) |
| channel | official-redirect (dl.snipaste.com/linux → latest AppImage) |

Snipaste — screenshot & paste tool (F1 snip / F3 paste). The official Linux build is a portable AppImage behind a stable "latest" redirect; this component puts it on the desktop and starts it with the session.

## Guard (idempotency)

```bash
test -x /root/Desktop/Snipaste.AppImage && test -f /root/.config/autostart/snipaste.desktop && echo present
```

## Install

```bash
export DEBIAN_FRONTEND=noninteractive
mkdir -p /root/Desktop /root/.config/autostart /root/.local/share/applications

# Official stable "latest" redirect (302). Never pin a version.
REDIR=$(curl -sSI --max-time 20 https://dl.snipaste.com/linux | grep -i '^location:' | tr -d '\r' | awk '{print $2}')
VERSION=$(echo "$REDIR" | grep -oP 'Snipaste-\K[0-9][0-9.]*')   # real app version — record this

curl --fail --location --retry 3 -o /root/Desktop/Snipaste.AppImage https://dl.snipaste.com/linux
chmod 755 /root/Desktop/Snipaste.AppImage

# AppImage type 2 runtime needs libfuse2 (present on most Ubuntu images)
ldconfig -p | grep -q libfuse.so.2 || apt-get install -y libfuse2t64

cat > /root/.config/autostart/snipaste.desktop <<'EOF'
[Desktop Entry]
Type=Application
Name=Snipaste
Comment=Snipaste screenshot & paste tool
Exec=/root/Desktop/Snipaste.AppImage
Terminal=false
X-GNOME-Autostart-enabled=true
EOF

cat > /root/.local/share/applications/snipaste.desktop <<'EOF'
[Desktop Entry]
Type=Application
Name=Snipaste
Comment=Snipaste screenshot & paste tool
Exec=/root/Desktop/Snipaste.AppImage
Terminal=false
Categories=Utility;Graphics;
EOF
```

## Verify

```bash
test -x /root/Desktop/Snipaste.AppImage && test -f /root/.config/autostart/snipaste.desktop
```

## Known Pitfalls

- `--appimage-version` prints the AppImage **runtime build id** (e.g. `8bbf694`), not the app version — parse the real version from the redirect `Location` at install time (verified 2026-10-06).
- If `libfuse2` is genuinely unavailable, install `libfuse2t64` (24.04+/26.04 package name) or `libfuse2`; as a last resort append `--appimage-extract-and-run` to `Exec` (extracts to /tmp on every launch, slower start, works without FUSE).
- Autostart fires at session **login** — an already-open RDP session won't show it until the next login, or launch it from the desktop icon / menu.
- F1 = screenshot, F3 = paste. The tray icon needs a system tray on the panel (Xfce's default panel has one). UI language is switchable in Snipaste preferences.
- Per-user config lands in `~/.config/Snipaste` (root here) — copying that directory from an existing machine migrates settings.

## Manual follow-ups

- none
