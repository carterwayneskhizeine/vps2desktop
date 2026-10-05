# Google Chrome

| | |
|---|---|
| id | `chrome` |
| group | apps |
| deps | none (pair with `xfce-desktop` for actual use) |
| channel | vendor-deb (always the current build) |

Latest `google-chrome-stable` from Google's own repo, plus a root-safe wrapper.

## Guard (idempotency)

```bash
google-chrome --version >/dev/null 2>&1 && test -x /usr/local/bin/google-chrome-stable && echo present
```

## Install

```bash
export DEBIAN_FRONTEND=noninteractive
cd /tmp
curl --fail --location --retry 3 -o chrome.deb https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
apt-get install -y ./chrome.deb   # resolves Google's apt repo + signing key as a side effect
rm -f chrome.deb

# Root sessions crash without --no-sandbox. /usr/local/bin shadows /usr/bin in PATH:
cat > /usr/local/bin/google-chrome-stable <<'EOF'
#!/bin/sh
exec /opt/google/chrome/chrome --no-sandbox "$@"
EOF
chmod 755 /usr/local/bin/google-chrome-stable

# Menu entry override so panel/taskbar launches also get the flag
mkdir -p /root/.local/share/applications
cp /usr/share/applications/google-chrome.desktop /root/.local/share/applications/
sed -i 's#Exec=/usr/bin/google-chrome-stable#Exec=/usr/local/bin/google-chrome-stable#' /root/.local/share/applications/google-chrome.desktop
```

## Verify

```bash
google-chrome --version && head -1 /usr/local/bin/google-chrome-stable | grep -q no-sandbox
```

## Known Pitfalls

- The `_current_amd64.deb` URL always fetches the newest stable build — never pin a Chrome version.
- Without the wrapper, Chrome opens and immediately dies on root ("Running as root without --no-sandbox is not supported").
- The wrapper works because PATH resolves `/usr/local/bin` before `/usr/bin` — do not "fix" the duplicate name.
