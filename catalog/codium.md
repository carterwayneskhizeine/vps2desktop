# VSCodium

| | |
|---|---|
| id | `codium` |
| group | apps |
| deps | none (pair with `xfce-desktop` for actual use) |
| channel | snap (classic, latest/stable) |

VSCodium (MIT-licensed VS Code build without Microsoft telemetry/branding) via snap, plus a root-safe wrapper. The snap's vendor launcher carries the same uid=0 guard as the `vscode` deb (c17).

## Guard (idempotency)

```bash
snap list codium >/dev/null 2>&1 && test -x /usr/local/bin/codium && echo present
```

## Install

```bash
snap install codium --classic

# Root sessions need --user-data-dir to pass the vendor launcher's uid=0 guard
# (--no-sandbox kept per vscode c17 practice). The wrapper must exec
# /snap/bin/codium by absolute path — exec'ing bare `codium` recurses into itself.
cat > /usr/local/bin/codium <<'EOF'
#!/bin/sh
exec /snap/bin/codium --no-sandbox --user-data-dir=/root/.config/VSCodium "$@"
EOF
chmod 755 /usr/local/bin/codium

# Menu overrides (snapd-managed entries; both must be covered). The url-handler
# snap command folds into the same wrapper: --open-url passes through "$@".
mkdir -p /root/.local/share/applications
for f in codium_codium.desktop codium_url-handler.desktop; do
  cp "/var/lib/snapd/desktop/applications/$f" "/root/.local/share/applications/$f" 2>/dev/null || continue
  sed -i 's#Exec=/snap/bin/codium.url-handler#Exec=/usr/local/bin/codium#; s#Exec=/snap/bin/codium#Exec=/usr/local/bin/codium#' "/root/.local/share/applications/$f"
done
```

## Verify

```bash
codium --version | head -1 && grep -q 'user-data-dir' /usr/local/bin/codium
```

## Known Pitfalls

- The root guard lives in `/snap/codium/<rev>/usr/share/codium/bin/codium` and its error text is misleading: it recommends `--no-sandbox` AND `--user-data-dir`, but the check only passes on `--user-data-dir`, `--user-data-dir=*`, `--file-write`, `tunnel` or `serve-web`. Same trap as vscode c17.
- The guard sits in the CLI script itself, so even `codium --version` exits 1 as root without flags — version checks must go through the wrapper.
- Snap menu entries live in `/var/lib/snapd/desktop/applications/` (not `/usr/share/applications/`), named `codium_codium.desktop` / `codium_url-handler.desktop`. Per-user copies in `/root/.local/share/applications/` win by XDG precedence.
- After a snap refresh, snapd may regenerate the vendor entries; if the menu stops launching through the wrapper, re-run the Install loop (same drift caveat as vscode's sed override).
- Under XRDP the verbose log shows `MESA-LOADER: failed to open swrast` and ANGLE `Could not create a backing OpenGL context` errors. Non-fatal — the window renders via software fallback; do not add `--disable-gpu` in response.
- With an instance already running, a new `codium` invocation hands off to it and exits 0 within ~2 s. Normal IPC forwarding, not a crash — confusing during launch testing.
- All pitfalls verified live on light-node-01 on 2026-10-14.
