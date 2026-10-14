# Notepad++ (Linux port)

| | |
|---|---|
| id | `notepadplusplus` |
| group | apps |
| deps | none (Qt6 runtime pulled by apt) |
| channel | github-api (release ships a single amd64 deb) |

Native Qt6/QScintilla port of Notepad++ (no Wine) — `mixelpixx/notepad-plus-plus-linux`. Installed from the GitHub release **deb**, never the snap (see Pitfalls).

## Guard (idempotency)

```bash
dpkg -s notepadplusplus >/dev/null 2>&1 && ! ldd /usr/bin/notepadplusplus | grep -q 'not found' && echo present
```

## Install

```bash
export DEBIAN_FRONTEND=noninteractive
DEB_URL=$(curl -fsSL --max-time 20 https://api.github.com/repos/mixelpixx/notepad-plus-plus-linux/releases/latest \
  | grep -oE '"browser_download_url":\s*"\K[^"]+\.deb')

cd /tmp
curl --fail --location --retry 3 -o npp.deb "$DEB_URL"
apt-get install -y ./npp.deb

# The deb does NOT declare libqt6xml6t64 (verified 1.2.0) — without it the binary
# dies at launch with libQt6Xml.so.6 not found. Install explicitly:
apt-get install -y libqt6xml6t64
rm -f npp.deb
```

## Verify

```bash
dpkg -s notepadplusplus | grep -q 'Status: install ok installed' \
  && ! ldd /usr/bin/notepadplusplus | grep -q 'not found' && echo ok
```

## Known Pitfalls

- **Do not install the `notepadplusplus` snap instead.** It is strict-confined: `/root` is readable but NOT writable from inside the sandbox (verified live — saves fail with Permission denied), which on root-managed machines makes it a viewer only. The deb has no such limit.
- The deb's `Depends` omits `libqt6xml6t64` — same undeclared-runtime trap as peazip's GTK2. `ldd /usr/bin/notepadplusplus | grep 'not found'` must be empty after install.
- No root guard (Qt app, not Electron): launches as root with no wrapper; the menu entry `notepadplusplus.desktop` works as-is (`Exec=notepadplusplus %F`).
- First run logs `Macro file does not exist` and `No translation found for "System" ( "C" )` — benign; it initializes `/root/.config/notepad-plus-plus/`.
- Upstream is young (single release at cataloging time, 2026-05-30); syntax highlighting covers 14 languages — far fewer than Windows Notepad++.
- Verified live on light-node-01 on 2026-10-14.
