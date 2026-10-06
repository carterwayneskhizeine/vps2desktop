# Catalog changelog

Append one entry per catalog change, newest last. Each entry has a stable id
(`c1`, `c2`, …) — manifests record the newest id they have seen in `last_sync`,
and update-check reads everything after it.

## c1 — 2026-10-05 — Initial catalog v1

Added components: base-tools, fail2ban, xfce-desktop, xrdp,
xrdp-audio, fcitx5-chinese, chrome, vscode, node-nvm, uv, anaconda,
claude-code, cc-switch-cli, aichat, codex, opencode, pi, agent-browser,
tavily-cli.

Presets: minimal, remote-desktop, ai-toolkit, full.

## c2 — 2026-10-06 — fixes from a live status pass (light-node-01)

- node-nvm / codex / opencode / pi / agent-browser (+ SKILL.md): the recommended
  `export PATH=/root/.nvm/versions/node/*/bin:$PATH` never worked — bash does not
  glob inside a `PATH=` assignment, so the literal `*` stayed in PATH and node/npm
  CLIs were silently not found. Use `for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done`.
- chrome: wrapper now appends `--no-sandbox` for uid 0 only (the same wrapper works
  from non-root sessions); Verify greps the whole wrapper instead of only its first line.
- agent-browser: 0.38+ stores browsers in `~/.agent-browser/browsers`; the Guard
  accepts that path as well as the old `~/.cache/agent-browser`.

## c3 — 2026-10-06 — fixes from the goldie08 deploy (Ubuntu 26.04)

- vscode: the documented download URL `update.code.visualstudio.com/latest/linux-deb-x64`
  was retired upstream (404). Use the official
  `code.visualstudio.com/sha/download?build=stable&os=linux-deb-x64` redirect instead.
- anaconda: upstream no longer publishes `.sha256` sidecar files in the archive at all
  (every release 404s). Checksum step removed — integrity relies on TLS to the official
  domain (user-approved). Installer lookup now uses `sort -uV` (each filename appears
  twice in the listing HTML; plain `-V` makes "fall back one version" resolve to itself).
- opencode / pi: npm ≥ 12 blocks their postinstall scripts — the install succeeds with
  only a warning and the CLI is broken until rerun with `--allow-scripts=<pkg>`
  (verified live; codex and agent-browser docs already carried this note).

## c4 — 2026-10-06 — xrdp unreachable behind image-default ufw (goldie08)

- xrdp: Vultr's Ubuntu 26.04 image ships ufw **enabled, default deny incoming, only 22
  allowed**. Every install-time Verify passes and the port shows LISTEN, but RDP is
  dropped from outside until `ufw allow 3389/tcp` (+udp) or a security-group rule is
  added — a local check cannot see this; probe from an external host. Documented as a
  Known Pitfall; the open-vs-source-restricted choice stays with the user.

## c5 — 2026-10-06 — new component: snipaste

- New `snipaste` (apps): official always-latest Linux AppImage via the
  `dl.snipaste.com/linux` 302 redirect, installed to `/root/Desktop/Snipaste.AppImage`
  with an Xfce autostart entry and a menu entry (verified live on goldie08, 2.11.3).
  Note: `--appimage-version` reports the AppImage runtime build id, not the app version —
  parse the version from the redirect Location instead.
- checklist.html: added snipaste; fixed the stale anaconda description
  ("checksum-verified" — obsolete since c3).

## c6 — 2026-10-06 — new component: peazip

- New `peazip` (apps): official GTK2 deb resolved via the GitHub releases API
  (asset names embed the version), plus Thunar custom actions for the right-click menu —
  Extract Here / Extract To... / Open in PeaZip / Add to ZIP / Add to 7Z
  (verified live on goldie08, 11.3.0). GTK2 variant chosen over Qt6 to keep the GTK
  desktop Qt-free; `apt` pulls `libgtk2.0-0t64` automatically when missing.
- Pitfall documented: Thunar caches custom actions — a running Thunar needs a restart
  (or relogin) before new uca.xml entries appear; empty elements in uca.xml mean
  "show for this type", missing ones mean "don't".

## c7 — 2026-10-06 — new component: warp-terminal

- New `warp-terminal` (apps): official GPG-signed apt repository at
  releases.warp.dev/linux/deb (`stable` channel; Warp Preview/dev deliberately not used),
  so `apt upgrade` tracks the latest — no AppImage (verified live on goldie08,
  0.2026.09.30.08.29.stable.01).
- Pitfall documented: the GitHub repo warpdotdev/Warp tags releases without assets —
  the apt repo is the real channel. GPU-rendered UI falls back to Mesa llvmpipe under
  RDP (libgl1-mesa-dri required). First-launch login is manual.
