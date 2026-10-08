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

## c8 — 2026-10-06 — chrome: desktop 🌐 icon fails as root (exo chain bypasses wrapper)

- chrome: the Xfce desktop Web-Browser icon and link-clicks resolve through exo helpers
  (name `google-chrome` via PATH) and the `x-www-browser` alternative (absolute symlink
  chain that bypasses the `/usr/local/bin` shadow). Bare chrome as root dies instantly →
  "Failed to execute default Web Browser (input/output error)". Fix: install a second
  root-safe wrapper named `google-chrome` and repoint the alternative to it
  (verified live on goldie08 via `exo-open --launch WebBrowser` in the RDP session).
- Pitfall noted: testing GUI launches from SSH requires detaching the child
  (`setsid ... </dev/null >/dev/null 2>&1`) — it inherits the SSH stdout pipe otherwise
  and the session never returns.

## c9 — 2026-10-06 — peazip: deb does not declare its GTK2 runtime dependency

- peazip: the official deb installs fine but does NOT declare `libgtk2.0-0t64`; on
  GTK3-only boxes the binary dies at launch with "error while loading shared libraries:
  libgdk-x11-2.0.so.0" (hit live on goldie08/26.04). Install now adds libgtk2.0-0t64
  explicitly, and Verify asserts `ldd` finds no missing libraries (c6's "apt pulls it
  automatically" assumption was wrong).
- Also corrected the SSH GUI-test technique: `setsid` execs into the child — it must be
  backgrounded (`setsid ... &`) or the SSH script waits for the app forever.

## c10 — 2026-10-08 — new component: tailscale

- New `tailscale` (base): latest stable package from Tailscale's official signed
  Ubuntu apt repository, with `tailscaled` enabled at boot.
- Verify checks package, CLI and daemon independently of tailnet authorization;
  login remains a manual follow-up. Added to the Full preset and checklist.

## c11 — 2026-10-08 — Xfce status tray and Fcitx5 icon repair

- xfce-desktop: add a live-session procedure to ensure the top panel's built-in
  `systray` plugin without replacing existing panel items.
- fcitx5-chinese: document that working input switching does not imply a visible
  tray icon; repair the panel's StatusNotifier host before restarting Fcitx5.

## c12 — 2026-10-08 — Tailscale desktop integration

- tailscale: document the bundled Linux systray and local device web UI, using
  the live desktop session's D-Bus environment. Check icon/menu compatibility
  before enabling tray startup on root Xfce desktops; keep login manual.
- Add a desktop/menu launcher for the local management UI and freedesktop tray
  autostart after verification; use the panel-appropriate icon theme.

## c13 — 2026-10-08 — new component: docker-desktop

- New `docker-desktop` (apps): latest official Docker Desktop Ubuntu amd64 DEB,
  installed from Docker's documented package URL. Adds Docker's signed apt source,
  QEMU, and `gnome-terminal` prerequisites. Documented first-run agreement and
  nested-virtualization limitation.

## c14 — 2026-10-08 — new component: docker-engine

- New `docker-engine` (apps): official Docker Engine, CLI, Buildx, and Compose V2
  plugin from Docker's signed Ubuntu apt repository. Verify the running daemon and
  run `hello-world` before recording the component.

## c15 — 2026-10-08 — new component: obsidian

- New `obsidian` (apps): resolve the latest official x86_64 AppImage from Obsidian's
  GitHub releases API, save it as `/root/Documents/Obsidian.AppImage`, extract its
  bundled icon, and create a trusted desktop launcher with `--no-sandbox` for the
  root Xfce session.
- Depends on `base-tools` and `xfce-desktop`; installs the AppImage FUSE runtime
  and SquashFS tools needed to run the portable app and extract only its icon.

## c16 — 2026-10-08 — snipaste: Documents location and desktop integration

- Update `snipaste` to save the latest official AppImage under `/root/Documents`,
  extract its bundled icon, and create a trusted desktop launcher. Keep its Xfce
  menu entry and session autostart, updating both to the stable Documents path.
- Add `base-tools` and `xfce-desktop` dependencies for release resolution, icon
  extraction and the graphical desktop integrations.
