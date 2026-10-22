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
  | grep -oP '"browser_download_url":\s*"\K[^"]+\.deb')

cd /tmp
curl --fail --location --retry 3 -o npp.deb "$DEB_URL"
apt-get install -y ./npp.deb

# The deb does NOT declare libqt6xml6t64 (verified 1.2.0) — without it the binary
# dies at launch with libQt6Xml.so.6 not found. Install explicitly:
apt-get install -y libqt6xml6t64
rm -f npp.deb
```

**On Ubuntu 26.04 the Qt6 runtime package names lost the `t64` suffix**, and
the release deb still depends on the 24.04 names. `apt-get install ./npp.deb`
then fails with `libqt6widgets6t64 ... not installable`. Repack the deb with
the 26.04 names instead of forcing the install:

```bash
cd /tmp
curl --fail --location --retry 3 -o npp.deb \
  https://github.com/mixelpixx/notepad-plus-plus-linux/releases/download/v1.2.0/notepadplusplus-1.2.0-Linux.deb

mkdir -p npp-pkg
dpkg-deb -R npp.deb npp-pkg
sed -i 's/libqt6widgets6t64/libqt6widgets6/' npp-pkg/DEBIAN/control
dpkg-deb -b npp-pkg npp-fixed.deb
apt-get install -y ./npp-fixed.deb

# The Qt6 XML runtime also lost the t64 suffix in 26.04:
apt-get install -y libqt6xml6
apt-get install -y libqscintilla2-qt6-15
rm -rf npp.deb npp-fixed.deb npp-pkg
```

`libqt6core6t64` still exists in 26.04 and needs no rewrite; only
`libqt6widgets6t64` and `libqt6xml6t64` were renamed.

## Verify

```bash
dpkg -s notepadplusplus | grep -q 'Status: install ok installed' \
  && ! ldd /usr/bin/notepadplusplus | grep -q 'not found' && echo ok
```

## Known Pitfalls

- **Do not install the `notepadplusplus` snap instead.** It is strict-confined: `/root` is readable but NOT writable from inside the sandbox (verified live — saves fail with Permission denied), which on root-managed machines makes it a viewer only. The deb has no such limit.
- The deb's `Depends` omits `libqt6xml6t64` — same undeclared-runtime trap as peazip's GTK2. `ldd /usr/bin/notepadplusplus | grep 'not found'` must be empty after install.
- **`grep -oE` does not support `\K`** — that is a PCRE construct. GNU grep
  only honours `\K` under `grep -P`. The Install's URL extraction silently
  produced an empty string and `curl` then failed with "blank argument".
  Every other catalog doc that uses `\K` already uses `-oP`; this one did not.
- **The deb's `Depends` names the 24.04 Qt6 packages** (`libqt6widgets6t64`),
  which do not exist in 26.04. Repack the deb with the 26.04 names rather than
  forcing the install with `--force-depends` — the renamed packages carry the
  same sonames, so nothing else changes.
- No root guard (Qt app, not Electron): launches as root with no wrapper; the menu entry `notepadplusplus.desktop` works as-is (`Exec=notepadplusplus %F`).
- First run logs `Macro file does not exist` and `No translation found for "System" ( "C" )` — benign; it initializes `/root/.config/notepad-plus-plus/`.
- Upstream is young (single release at cataloging time, 2026-05-30); syntax highlighting covers 14 languages — far fewer than Windows Notepad++.
- Verified live on light-node-01 on 2026-10-14.
