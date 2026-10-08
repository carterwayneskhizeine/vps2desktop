# Snipaste

| | |
|---|---|
| id | `snipaste` |
| group | apps |
| deps | `base-tools`, `xfce-desktop` |
| channel | official-redirect (dl.snipaste.com/linux → latest x86_64 AppImage) |
| upstream | https://www.snipaste.com/ |

Snipaste — screenshot & paste tool (F1 snip / F3 paste). The official Linux build is a portable AppImage. This component saves it to `/root/Documents`, extracts the bundled icon, creates a trusted desktop launcher, and configures the menu and session autostart.

## Guard (idempotency)

```bash
test -x /root/Documents/Snipaste.AppImage \
  && test -s /root/.local/share/icons/snipaste.png \
  && test -s /root/.local/share/snipaste-version \
  && test -x /root/Desktop/Snipaste.desktop \
  && grep -q '^Exec=/root/Documents/Snipaste.AppImage$' /root/Desktop/Snipaste.desktop \
  && grep -q '^Icon=/root/.local/share/icons/snipaste.png$' /root/Desktop/Snipaste.desktop \
  && gio info -a metadata::trusted /root/Desktop/Snipaste.desktop | grep -q 'metadata::trusted: true' \
  && test -f /root/.config/autostart/snipaste.desktop \
  && test -f /root/.local/share/applications/snipaste.desktop \
  && echo present
```

## Install

Run in the target root Xfce desktop environment after confirming the machine identity. Resolve the current version from the official redirect; never pin a version.

```bash
set -euo pipefail
source /etc/os-release
test "$ID" = ubuntu
test "$(dpkg --print-architecture)" = amd64

DEBIAN_FRONTEND=noninteractive apt-get install -y squashfs-tools
if ! ldconfig -p | grep -q 'libfuse.so.2'; then
  DEBIAN_FRONTEND=noninteractive apt-get install -y libfuse2t64 \
    || DEBIAN_FRONTEND=noninteractive apt-get install -y libfuse2
fi

REDIR=$(curl --fail --silent --show-error --head --location --max-time 20 \
  --output /dev/null --write-out '%{url_effective}' https://dl.snipaste.com/linux)
case "$REDIR" in
  https://download.snipaste.com/archives/Snipaste-*-x86_64.AppImage) ;;
  *) printf 'Unexpected official redirect target\n' >&2; exit 1;;
esac
VERSION=$(printf '%s' "$REDIR" | grep -oP 'Snipaste-\K[0-9][0-9.]*')
test -n "$VERSION"

mkdir -p /root/Documents /root/Desktop /root/.local/share/icons \
  /root/.local/share/applications /root/.config/autostart
curl --fail --location --retry 3 --output /root/Documents/Snipaste.AppImage "$REDIR"
chmod 755 /root/Documents/Snipaste.AppImage

# Extract only Snipaste's bundled PNG icon from the AppImage SquashFS image.
OFFSET=$(/root/Documents/Snipaste.AppImage --appimage-offset)
case "$OFFSET" in ''|*[!0-9]*) exit 1;; esac
unsquashfs -offset "$OFFSET" -cat /root/Documents/Snipaste.AppImage \
  Snipaste.png > /root/.local/share/icons/snipaste.png
test -s /root/.local/share/icons/snipaste.png

cat > /root/Desktop/Snipaste.desktop <<'EOF'
[Desktop Entry]
Version=1.0
Type=Application
Name=Snipaste
Comment=Screenshot and paste tool
Exec=/root/Documents/Snipaste.AppImage
Icon=/root/.local/share/icons/snipaste.png
Terminal=false
Categories=Utility;Graphics;
EOF
chmod 755 /root/Desktop/Snipaste.desktop
gio set /root/Desktop/Snipaste.desktop metadata::trusted true

cat > /root/.config/autostart/snipaste.desktop <<'EOF'
[Desktop Entry]
Type=Application
Name=Snipaste
Comment=Screenshot and paste tool
Exec=/root/Documents/Snipaste.AppImage
Icon=/root/.local/share/icons/snipaste.png
Terminal=false
X-GNOME-Autostart-enabled=true
EOF

cat > /root/.local/share/applications/snipaste.desktop <<'EOF'
[Desktop Entry]
Type=Application
Name=Snipaste
Comment=Screenshot and paste tool
Exec=/root/Documents/Snipaste.AppImage
Icon=/root/.local/share/icons/snipaste.png
Terminal=false
Categories=Utility;Graphics;
EOF
printf '%s\n' "$VERSION" > /root/.local/share/snipaste-version
```

## Verify

```bash
test -x /root/Documents/Snipaste.AppImage
test -s /root/.local/share/icons/snipaste.png
test "$(od -An -tx1 -N8 /root/.local/share/icons/snipaste.png | tr -d ' \n')" = 89504e470d0a1a0a
test -s /root/.local/share/snipaste-version
test -x /root/Desktop/Snipaste.desktop
grep -q '^Exec=/root/Documents/Snipaste.AppImage$' /root/Desktop/Snipaste.desktop
grep -q '^Icon=/root/.local/share/icons/snipaste.png$' /root/Desktop/Snipaste.desktop
gio info -a metadata::trusted /root/Desktop/Snipaste.desktop | grep -q 'metadata::trusted: true'
test -f /root/.config/autostart/snipaste.desktop
test -f /root/.local/share/applications/snipaste.desktop
OFFSET=$(/root/Documents/Snipaste.AppImage --appimage-offset)
case "$OFFSET" in ''|*[!0-9]*) exit 1;; esac
cat /root/.local/share/snipaste-version
```

## Known Pitfalls

- The official redirect target embeds the Snipaste version. Resolve it at install time; do not hardcode a release or substitute a third-party build.
- The AppImage needs the FUSE 2 compatibility library. On Ubuntu 24.04+ this is normally `libfuse2t64`; older Ubuntu releases may package it as `libfuse2`. `--appimage-extract-and-run` can be added to the launcher's `Exec` as a slower fallback if FUSE is unavailable.
- The desktop launcher must be executable and trusted by GIO for Xfce to open it on double-click. If the desktop still prompts, use its “Allow Launching” action.
- Autostart runs at the next session login; an already-open RDP session will not start Snipaste until you launch it manually or log in again.
- The tray icon needs a system tray in the Xfce panel. F1 starts a screenshot and F3 pastes the last capture.
- AppImage auto-updates do not replace the downloaded base AppImage. Periodically fetch the latest build from the official redirect and keep the installed version record in sync.
- Per-user settings are in `/root/.config/Snipaste` for this root desktop profile.

## Manual follow-ups

- none
