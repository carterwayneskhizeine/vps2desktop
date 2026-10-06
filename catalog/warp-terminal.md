# Warp terminal

| | |
|---|---|
| id | `warp-terminal` |
| group | apps |
| deps | none (pair with `xfce-desktop` for actual use) |
| channel | apt (official releases.warp.dev repo, `stable`) |

Warp — the GPU-accelerated Rust terminal with built-in AI. Installed from Warp's **official apt repository** (`stable` channel), so `apt upgrade` keeps it current; GPG-signed, no AppImage involved.

## Guard (idempotency)

```bash
command -v warp-terminal >/dev/null 2>&1 && test -f /etc/apt/sources.list.d/warpdotdev.list && echo present
```

## Install

```bash
export DEBIAN_FRONTEND=noninteractive
# 1. Signing key
curl --fail --location --retry 3 -sS https://releases.warp.dev/linux/keys/warp.asc | gpg --dearmor > /tmp/warpdotdev.gpg
install -D -o root -g root -m 644 /tmp/warpdotdev.gpg /etc/apt/keyrings/warpdotdev.gpg
rm -f /tmp/warpdotdev.gpg

# 2. Repo (stable channel — Warp Preview/dev is a separate channel, do not use)
echo 'deb [arch=amd64 signed-by=/etc/apt/keyrings/warpdotdev.gpg] https://releases.warp.dev/linux/deb stable main' \
  > /etc/apt/sources.list.d/warpdotdev.list
apt-get update

# 3. Package + software GL provider (no GPU under RDP -> Mesa llvmpipe)
apt-get install -y warp-terminal
dpkg -s libgl1-mesa-dri >/dev/null 2>&1 || apt-get install -y libgl1-mesa-dri
```

## Verify

```bash
dpkg-query -W -f='${Version}\n' warp-terminal && warp-terminal --version
```

(`--version` works headless — prints `Oz v<version>...`.)

## Known Pitfalls

- The GitHub repo `warpdotdev/Warp` publishes release **tags without assets** (it is the issue tracker) — the apt repo above is the real distribution channel; don't chase GitHub download URLs.
- Package and binary are both `warp-terminal` (not `warp`); the menu entry is "Warp".
- Warp renders on the GPU. Under xrdp there is none — it falls back to Mesa software rendering (llvmpipe), which needs `libgl1-mesa-dri`; expect slower first paint than on a GPU box, typing stays usable.
- **Login is manual**: first launch asks for a Warp account (browser OAuth). Never automate it.
- The `.deb` from warp.dev does the same repo setup automatically — the manual steps above just make it explicit and idempotent.

## Manual follow-ups

- `warp-terminal` → account login on first use (OAuth).
- Optional: set Warp as the default terminal in Xfce → Preferred Applications.
