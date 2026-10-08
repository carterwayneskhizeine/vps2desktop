# Obsidian

| | |
|---|---|
| id | `obsidian` |
| group | apps |
| deps | `base-tools`, `xfce-desktop` |
| channel | github-api (latest official Linux x86_64 AppImage) |
| upstream | https://obsidian.md/download |

Obsidian is installed as a portable AppImage at `/root/Documents/Obsidian.AppImage`. The component extracts the bundled application icon and creates a trusted desktop launcher. Root sessions need `--no-sandbox` when launching the Electron app.

## Guard (idempotency)

```bash
test -x /root/Documents/Obsidian.AppImage \
  && test -s /root/.local/share/icons/obsidian.png \
  && test -s /root/.local/share/obsidian-version \
  && test -x /root/Desktop/Obsidian.desktop \
  && grep -q '^Exec=/root/Documents/Obsidian.AppImage --no-sandbox$' /root/Desktop/Obsidian.desktop \
  && grep -q '^Icon=/root/.local/share/icons/obsidian.png$' /root/Desktop/Obsidian.desktop \
  && echo present
```

## Install

Resolve the current release from Obsidian's official GitHub releases API; never pin a version. Run in the target root desktop session after confirming the machine identity.

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

RELEASE=$(curl --fail --location --retry 3 https://api.github.com/repos/obsidianmd/obsidian-releases/releases/latest)
TAG=$(printf '%s' "$RELEASE" | jq -r '.tag_name')
VERSION=${TAG#v}
APP_URL=$(printf '%s' "$RELEASE" | jq -r --arg name "Obsidian-$VERSION.AppImage" '.assets[] | select(.name == $name) | .browser_download_url')
test -n "$TAG" && test "$APP_URL" != null && test -n "$APP_URL"

mkdir -p /root/Documents /root/Desktop /root/.local/share/icons
curl --fail --location --retry 3 --output /root/Documents/Obsidian.AppImage "$APP_URL"
chmod 755 /root/Documents/Obsidian.AppImage

# Extract only the official icon from the AppImage's SquashFS image.
OFFSET=$(/root/Documents/Obsidian.AppImage --appimage-offset)
case "$OFFSET" in ''|*[!0-9]*) exit 1;; esac
unsquashfs -offset "$OFFSET" -cat /root/Documents/Obsidian.AppImage \
  usr/share/icons/hicolor/512x512/apps/obsidian.png \
  > /root/.local/share/icons/obsidian.png
test -s /root/.local/share/icons/obsidian.png

cat > /root/Desktop/Obsidian.desktop <<'EOF'
[Desktop Entry]
Version=1.0
Type=Application
Name=Obsidian
Comment=Open Obsidian
Exec=/root/Documents/Obsidian.AppImage --no-sandbox
Icon=/root/.local/share/icons/obsidian.png
Terminal=false
Categories=Office;
EOF
chmod 755 /root/Desktop/Obsidian.desktop
gio set /root/Desktop/Obsidian.desktop metadata::trusted true
printf '%s\n' "$VERSION" > /root/.local/share/obsidian-version
```

## Verify

```bash
test -x /root/Documents/Obsidian.AppImage
test -s /root/.local/share/icons/obsidian.png
test -s /root/.local/share/obsidian-version
test "$(od -An -tx1 -N8 /root/.local/share/icons/obsidian.png | tr -d ' \n')" = 89504e470d0a1a0a
grep -q '^Exec=/root/Documents/Obsidian.AppImage --no-sandbox$' /root/Desktop/Obsidian.desktop
grep -q '^Icon=/root/.local/share/icons/obsidian.png$' /root/Desktop/Obsidian.desktop
OFFSET=$(/root/Documents/Obsidian.AppImage --appimage-offset)
case "$OFFSET" in ''|*[!0-9]*) exit 1;; esac
cat /root/.local/share/obsidian-version
```

## Known Pitfalls

- The GitHub asset name contains its release version. Resolve it from the latest-release API response and do not hardcode the version.
- The AppImage is the official Linux download format; do not substitute community repacks.
- A root Electron process exits without `--no-sandbox`; keep the option in the desktop launcher's `Exec` line.
- The `.desktop` launcher must be executable and trusted by GIO for Xfce to launch it on double-click. If the desktop environment still prompts, use its “Allow Launching” action.
- AppImage auto-updates do not update the base installer/Electron runtime. Periodically download the latest AppImage and replace `/root/Documents/Obsidian.AppImage`; the desktop launcher and icon paths remain stable.
- The latest public AppImage asset is x86_64. This component requires an Ubuntu amd64 target and an Xfce desktop session.

## Manual follow-ups

- Create or select a vault on first launch.
