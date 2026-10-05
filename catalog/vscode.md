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
for f in code.desktop code-url-handler.desktop; do
  cp "/usr/share/applications/$f" "/root/.local/share/applications/$f" 2>/dev/null || continue
  sed -i 's#Exec=/usr/share/code/code#Exec=/usr/local/bin/code#' "/root/.local/share/applications/$f"
done
```

## Verify

```bash
code --version | head -1 && grep -q 'user-data-dir' /usr/local/bin/code
```

## Known Pitfalls

- The `sha/download?build=stable&os=linux-deb-x64` URL always redirects to the current stable — never pin a VS Code build. (The old `update.code.visualstudio.com/latest/...` path was retired upstream and returns 404 — verified 2026-10-06.)
- Headless note: `code --version` may print the version to stdout only sporadically over SSH (X/ozone errors go to stderr); `dpkg-query -W code` is the reliable version source for records.
- On root, missing `--user-data-dir` produces a broken/grey window even with `--no-sandbox` set — both flags are required.
