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
google-chrome --version >/dev/null 2>&1 && test -x /usr/local/bin/google-chrome-stable && test -x /usr/local/bin/google-chrome && echo present
```

## Install

```bash
export DEBIAN_FRONTEND=noninteractive
cd /tmp
curl --fail --location --retry 3 -o chrome.deb https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
apt-get install -y ./chrome.deb   # resolves Google's apt repo + signing key as a side effect
rm -f chrome.deb

# Root sessions crash without --no-sandbox. /usr/local/bin shadows /usr/bin in PATH.
# Append the flag for uid 0 only, so the same wrapper also works from non-root sessions:
cat > /usr/local/bin/google-chrome-stable <<'EOF'
#!/bin/sh
[ "$(id -u)" = 0 ] && set -- "$@" --no-sandbox
exec /opt/google/chrome/chrome "$@"
EOF
chmod 755 /usr/local/bin/google-chrome-stable

# Menu entry override so panel/taskbar launches also get the flag
mkdir -p /root/.local/share/applications
cp /usr/share/applications/google-chrome.desktop /root/.local/share/applications/
sed -i 's#Exec=/usr/bin/google-chrome-stable#Exec=/usr/local/bin/google-chrome-stable#' /root/.local/share/applications/google-chrome.desktop

# The Xfce desktop 🌐 icon / "open link" chain goes through exo helpers
# (helpers.rc: WebBrowser=google-chrome, resolved by NAME via PATH) and through
# the x-www-browser alternative (an absolute symlink chain that BYPASSES the
# /usr/local/bin shadow). Shadow the second name too, and repoint the alternative:
cat > /usr/local/bin/google-chrome <<'EOF'
#!/bin/sh
[ "$(id -u)" = 0 ] && set -- "$@" --no-sandbox
exec /opt/google/chrome/chrome "$@"
EOF
chmod 755 /usr/local/bin/google-chrome
update-alternatives --install /usr/bin/x-www-browser x-www-browser /usr/local/bin/google-chrome 1000
update-alternatives --set x-www-browser /usr/local/bin/google-chrome
```

## Verify

```bash
google-chrome --version && grep -q no-sandbox /usr/local/bin/google-chrome-stable && test -x /usr/local/bin/google-chrome
```

## Known Pitfalls

- The `_current_amd64.deb` URL always fetches the newest stable build — never pin a Chrome version.
- Without the wrapper, Chrome opens and immediately dies on root ("Running as root without --no-sandbox is not supported").
- The wrapper works because PATH resolves `/usr/local/bin` before `/usr/bin` — do not "fix" the duplicate name.
- **Desktop 🌐 icon vs menu**: the menu entry execs our wrapper, but the desktop "Web Browser" icon and link-clicks go through **exo** (`exo-open --launch WebBrowser`), which resolves the *name* `google-chrome` from `X-XFCE-Binaries` via PATH — while `/usr/bin/google-chrome` is an absolute symlink chain (`/etc/alternatives` → `/usr/bin/google-chrome-stable`) that bypasses PATH shadowing entirely. As root that bare binary dies instantly and exo reports `Failed to execute default Web Browser — input/output error`. Both the `google-chrome`-named wrapper and the repointed `x-www-browser` alternative are required (verified live 2026-10-06).
- To test a GUI launch from an SSH session, detach it or the GUI child inherits the SSH stdout pipe and the session hangs: `setsid exo-open --launch WebBrowser </dev/null >/dev/null 2>&1`.
