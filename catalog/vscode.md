# Visual Studio Code

| | |
|---|---|
| id | `vscode` |
| group | apps |
| deps | none (pair with `xfce-desktop` for actual use) |
| channel | vendor-deb (always the current stable) |

Latest stable VS Code from Microsoft's official download endpoint, plus a root-safe wrapper.

## Guard (idempotency)

```bash
code --version >/dev/null 2>&1 && test -x /usr/local/bin/code && echo present
```

## Install

```bash
export DEBIAN_FRONTEND=noninteractive
cd /tmp
curl --fail --location --retry 3 -o code.deb "https://code.visualstudio.com/sha/download?build=stable&os=linux-deb-x64"
apt-get install -y ./code.deb
rm -f code.deb

# Root sessions need --no-sandbox AND --user-data-dir (else: blank window / crash).
cat > /usr/local/bin/code <<'EOF'
#!/bin/sh
exec /usr/share/code/code --no-sandbox --user-data-dir=/root/.config/Code "$@"
EOF
chmod 755 /usr/local/bin/code

# Menu overrides (both desktop entries must be covered)
mkdir -p /root/.local/share/applications
for f in com.microsoft.VSCode.desktop com.microsoft.VSCode.UrlHandler.desktop; do
  cp "/usr/share/applications/$f" "/root/.local/share/applications/$f" 2>/dev/null || continue
  sed -i 's#Exec=/usr/share/code/code#Exec=/usr/local/bin/code#' "/root/.local/share/applications/$f"
done
```

## Verify

```bash
timeout 20 code --version 2>/dev/null | head -1; dpkg-query -W -f='${Version}\n' code
grep -q 'user-data-dir' /usr/local/bin/code
```

`code --version` does not exit on its own in a headless session — pipe it into
`head -1` (the closed pipe kills it) or wrap it in `timeout`, otherwise a
Verify loop hangs forever. `dpkg-query -W code` is the version of record.

## Known Pitfalls

- **The deb upgrade deletes the root wrapper.** `apt-get install -y ./code.deb` removes `/usr/local/bin/code` — the package ships its own `/usr/bin/code` and knows nothing about the wrapper. Observed on two machines (`light-node-01` 2026-10-11, `contabo` 2026-10-23). The menu entries then fall back to the bare binary as root, which gives a blank window or a crash. Re-create the wrapper and re-apply the desktop overrides after every upgrade — i.e. re-run the whole Install block, not just the download.
- **The download URL is newer than the vendor APT repo.** On 2026-10-23 the stable download served `1.143.1-1792691907` while `apt-cache policy code` offered `1.143.0-1792533444` as the candidate. `apt-get upgrade` therefore leaves you one build behind — always install from the download URL, never from the repo.
- The `sha/download?build=stable&os=linux-deb-x64` URL always redirects to the current stable — never pin a VS Code build. (The old `update.code.visualstudio.com/latest/...` path was retired upstream and returns 404 — verified 2026-10-06.)
- Headless note: `code --version` may print the version to stdout only sporadically over SSH (X/ozone errors go to stderr); `dpkg-query -W code` is the reliable version source for records. It also does not exit on its own — see Verify.
- On root, missing `--user-data-dir` produces a broken/grey window even with `--no-sandbox` set — both flags are required.
- Vendor .desktop filenames: the official deb currently ships `com.microsoft.VSCode.desktop` and `com.microsoft.VSCode.UrlHandler.desktop`. Older builds shipped `code.desktop` / `code-url-handler.desktop` — if those names come back or change again, update the Install loop above; the `|| continue` will otherwise silently no-op and the menu will launch the bare `/usr/bin/code` binary as root.
- The sed override assumes the vendor's `Exec=` line still points at `/usr/share/code/code`; if Microsoft repaths the entry, the override silently no-ops and the menu reverts to the bare binary. If the wrapper at `/usr/local/bin/code` is missing for any reason, the menu entry fails to launch — run Install again to restore it.
