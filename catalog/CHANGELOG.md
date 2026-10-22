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

## c17 — 2026-10-11 — vscode: .desktop override loop updated to current vendor filenames

- vscode: the Install loop's `for f in code.desktop code-url-handler.desktop` no
  longer matches the vendor .deb. Current vendor filenames are
  `com.microsoft.VSCode.desktop` and `com.microsoft.VSCode.UrlHandler.desktop`;
  the loop is updated, and the existing `sed` pattern still matches the vendor's
  `Exec=/usr/share/code/code` line.
- Two new Known Pitfalls added: vendor .desktop filenames have changed at least
  once already (revisit them after future VS Code releases); and the `sed`
  override silently no-ops if Microsoft repaths the `Exec=` target or if the
  `/usr/local/bin/code` wrapper is missing — both leave the menu launching the
  bare binary as root.
- Hit live on light-node-01 during a 1.140.0 → 1.141.0 upgrade: the wrapper was
  missing before the upgrade even ran, so the .desktop overrides pointed at a
  non-existent target. Catalog now records the recovery path.

## c18 — 2026-10-12 — tavily-cli: Known Pitfalls from a live light-node-01 pass

- `--country` requires a valid country name, not an ISO code — verified by a
  real search request, `CN` returns "Invalid country". Consult the API
  reference for the supported list before passing the flag.
- `tvly init [--no-browser]` is the official skill installer for agent hosts
  (Claude Code / Codex detected automatically); with `--no-browser` it falls
  back to the `TAVILY_API_KEY` env var. Bundled skills are pinned to the CLI
  release, so re-run after every CLI upgrade.
- `tvly login --api-key <key>` persists the API key to `~/.tavily/config.json`
  (mode 600); preferred over browser OAuth when an API key is already available,
  since browser OAuth routes through Tavily's MCP endpoint with documented
  compatibility gaps for some `search` and `research` options.
- `tvly auth` masks the displayed API key (e.g. `tvly-dev...H2H6`), a useful
  existence check that does not leak the full secret.
- All four findings verified live on light-node-01 on 2026-10-12.

## c19 — 2026-10-14 — new component: chatgpt

- New `chatgpt` (apps): install an explicitly user-selected local amd64 DEB with
  APT dependency resolution, preserving the requested package version.
- Add a root-safe command wrapper, per-user menu override and trusted desktop
  launcher. Verify package status, shared libraries and desktop entries, plus an
  actual GUI launch when an Xfce session is available. Login remains manual.
- The component requires a supplied package and is not included in the Full
  preset. Vendor repository configuration stays managed by the package.

## c20 — 2026-10-14 — new component: cyrus

- New `cyrus` (ai-cli): install `cyrus-ai@latest`, initialize config without
  overwriting existing credentials, and verify the CLI, JSON and permissions.
  Register dependencies and add the Component to the checklist, AI Toolkit and
  Full presets.
- Document the manual Linear OAuth app/authorization and GitHub repository flow,
  Docker-hosted Cloudflare Tunnel host-gateway routing, read-only functional
  checks and optional systemd persistence with resolved nvm paths.
- Capture signature-secret mismatch, missing Agent delegation, and hanging
  `check-tokens` diagnostics. Explain how Linear's team status categories differ
  from Agent Session completion, and retain machine-specific facts in Local State.

## c21 — 2026-10-14 — new component: codium

- New `codium` (apps): VSCodium via snap `--classic`, plus a root-safe wrapper
  injecting `--no-sandbox`/`--user-data-dir` — the snap's vendor launcher guards
  uid=0 exactly like the VS Code deb (c17), with the same misleading error text.
- Per-user menu overrides for the snapd-managed entries (`codium_codium.desktop`,
  `codium_url-handler.desktop`) redirect Exec to the wrapper; the url-handler
  snap command folds into the same wrapper via arg pass-through.
- Pitfalls captured: the guard only accepts `--user-data-dir`-class flags (even
  `codium --version` is blocked as root), benign MESA/ANGLE software-render
  errors under XRDP, and fast exit-0 forwarding when an instance is already
  running.
- Not added to any preset — coexists with `vscode` as a per-machine editor
  choice. All steps verified live on light-node-01.

## c22 — 2026-10-14 — new component: notepadplusplus

- New `notepadplusplus` (apps): native Qt6/QScintilla Linux port of Notepad++
  from the GitHub release deb (github-api channel, always latest). No root
  guard, no wrapper or menu override needed.
- The snap of the same name was installed first, then removed: it is
  strict-confined and `/root` is read-only inside the sandbox (saves fail with
  Permission denied) — documented as a do-not-use trap for root-managed
  machines.
- The deb's Depends omits `libqt6xml6t64`; installed explicitly, same
  undeclared-runtime trap pattern as peazip's GTK2 (c-series precedent).
- Verified live on light-node-01: ldd clean, 15 s GUI smoke run as root.

## c23 — 2026-10-16 — new component: bash-completion

- New `bash-completion` (base): apt `bash-completion` 1:2.11-8, giving the stock
  `/bin/bash` login shell Tab completion for 936 third-party CLIs (docker, git,
  systemctl, kubectl). No shell switch, no oh-my-zsh, nothing that affects
  running services.
- Installing the package is **not** sufficient on its own: Ubuntu's `~/.bashrc`
  ships the `. /etc/bash_completion` lines commented out. The component's Install
  step uncomments them; the Guard checks both halves.
- Verification trap documented: `complete -p <cmd>` returns "no completion
  specification" even when working, because the handler is registered as the
  *default* (`complete -D -F _completion_loader`) and command scripts load
  lazily on first Tab. `complete -D` also needs `-p`, or it performs a
  completion instead of printing one. Verify with `bash -ic`, since `~/.bashrc`
  returns early on `[ -z "$PS1" ]` for non-interactive shells.
- Chosen over zsh on light-node-01: zsh was installed and rolled back the same
  day (see the session notes). bash-completion keeps the shell and its script
  compatibility untouched; zsh remains the larger upgrade if ever revisited.
- Not added to any preset — same treatment as `codium` and `notepadplusplus`.
  Verified live on light-node-01 (Guard `active`, 936 completion scripts).

## c24 — 2026-10-17 — new component: webssh

- New `webssh` (apps, uv-tool channel): `uv tool install webssh` from PyPI
  `webssh`, whose `home_page` is exactly `https://github.com/huashengdun/webssh`
  — the user-specified upstream. Provides the `wssh` binary; no config file, no
  daemon package, no build step (Tornado + Paramiko + xterm.js).
- Deliberately **not** `pip install webssh` into the system interpreter: on these
  machines `python3` resolves to `/root/anaconda3/bin/python3`, so the install
  would drop webssh's pinned `paramiko`/`cryptography` wheels into the Anaconda
  environment. `uv tool` keeps it isolated under
  `/root/.local/share/uv/tools/webssh`.
- Deployed as a root systemd unit bound to `172.17.0.1:8127` (user-selected port,
  replacing the upstream default 8888), enabled at boot, fronted by the user's
  Cloudflare Tunnel container. Rebinding from `127.0.0.1` was required after the
  first live 502 — see below.
- **Security framing recorded in the component**: webssh has no authentication of
  its own, so any page it serves is an open SSH-credential form. `0.0.0.0` would
  publish that, and on this machine `ufw` is absent with an ACCEPT INPUT policy,
  so the wildcard would put it on the public IP. The unit binds a single address.
- **Headline pitfall — a loopback bind 502s behind a containerized Cloudflare
  Tunnel.** cloudflared runs in a container with `--add-host
  host.docker.internal:host-gateway` and therefore dials the Docker bridge gateway
  `172.17.0.1`, never `127.0.0.1`; the connection is refused by the kernel before
  webssh sees it (`dial tcp 172.17.0.1:8127: connect: connection refused`). The
  component's Install now derives the bind address from the terminator's location
  instead of hard-coding loopback, and its Verify curls the origin *and* the
  public tunnel hostname. Note the container sits on `172.19.0.0/16` while
  `host-gateway` still resolves to `172.17.0.1` — the default bridge gateway, not
  the tunnel network's; read it from the log rather than inferring it.
- **Sweep sibling routes when one 502s.** The same fault silently affected
  `aios-light.goldie-rill.top`, whose Vite dev server also binds `127.0.0.1` and
  had logged 8 identical refusals — it only looked healthy because nobody had
  requested it. `docker logs cloudflare-tunnel | grep -c 'connect: connection
  refused'` per origin is the quick triage.
- **A loopback bind is not a firewall** — recorded so the fix does not slide into
  `0.0.0.0` on the next machine.
- Other pitfalls captured from reading `handler.py` and installing live: `--fbidhttp`
  (default on) 403s plain HTTP from any source whose IP is not `is_private` —
  which excludes Tailscale's `100.64.0.0/10` CGNAT range, so a Tailscale-bound
  deployment over plain http fails; the origin check compares the `Origin` netloc
  to the `Host` header **including port**, so a proxy that rewrites `Host`
  (Cloudflare `http2Origin: true`, a bad `proxy_set_header Host $host`) yields
  "Cross origin operation is not allowed."; `--xheaders` default True makes
  tornado honour `X-Forwarded-Proto` and skip the plain-http branch entirely,
  which is why the tunnel path works; and `--policy` governs the *outbound* SSH
  session's host keys, not webssh's own access.
- Verification traps documented: asset URLs in the served page are relative
  (`static/js/main.js`), so guessed absolute paths such as `/xterm/css/xterm.css`
  404 and mimic a broken install; the WebSocket channel is proven by a `101`
  handshake on `/ws` rather than by a 200, and curl then hangs on the upgraded
  connection, so that timeout is expected rather than a failure.
- `--maxconn` is 20 live sessions **per client IP**, which behind a proxy that
  fronts every session from one bridge gateway is effectively global.
- **Usage section added** to the component after a live round-trip test: the
  Hostname field is **resolved by the server**, not the browser. webssh calls
  `socket.getaddrinfo()` on its own host, so a name that resolves on the user's
  laptop fails there with `socket.gaierror: [Errno -2] Name or service not known`,
  surfaced as `Unable to connect to <host>:<port>` — **before authentication**, so a
  correct password is still rejected and the message misleadingly reads as a
  credential problem. Test names with `getent hosts <name>` on the server; for the
  webssh host itself use `127.0.0.1` or the `/etc/hosts` alias `ubuntu` →
  `127.0.1.1`.
- **Tailscale MagicDNS rewrites addresses the user types**: `getent hosts
  38.60.204.237` returns `100.119.149.52 … light-node-01.tail2398a5.ts.net`, so a
  session intended for the public path can land on the tailnet interface.
- **`uv tool install` inherited Anaconda as the tool's base interpreter**
  (`bin/python` → `/root/anaconda3/bin/python3.14`). site-packages are isolated but
  the stdlib is not — tracebacks mix the venv's `paramiko` with
  `/root/anaconda3/lib/python3.14/socket.py` and `concurrent/futures/_base.py`.
  Removing or upgrading Anaconda would break webssh without naming the cause. The
  decoupling command (`uv tool install --python /usr/bin/python3 --force webssh`,
  dropping to 3.12) is documented; the user chose to leave the running install
  alone for now, so light-node-01 still carries the coupling.
- Not added to any preset — same treatment as `codium`, `notepadplusplus` and
  `bash-completion`. Verified live on light-node-01: webssh 1.6.3, tornado 6.5.10,
  paramiko 5.0.0 under Python 3.14; index, assets, `101` upgrade and an
  in-process paramiko handshake against the local sshd (ssh-ed25519,
  OpenSSH 9.6p1) all pass, and `https://webssh.goldie-rill.top/` returns 200 with
  no tunnel errors after the rebind.

## c25 — 2026-10-17 — chrome: pitfall — automation tools bypass the PATH wrapper

- `chrome`: documented that Puppeteer-based automation (notably `chrome-devtools-mcp`)
  resolves Chrome by absolute path (`/usr/bin/google-chrome` → `/etc/alternatives` →
  `/opt/google/chrome/google-chrome`) and never sees the `/usr/local/bin` wrapper,
  so root sessions still die with `crbug.com/638180` despite the wrapper being
  installed. Consumer-side fix: `--chromeArg=--no-sandbox` in the MCP server launch
  args (`--headless` too when no display is attached). Claude Code specifics that
  cost a debugging round: the plugin's effective launch config is the `mcpServers`
  block in `.claude-plugin/plugin.json`, not the root-level `mcp.json` (cross-tool
  spec — editing it changes nothing), and a plugin upgrade reinstalls into a new
  version dir, wiping the patch. Confirmed by `ps`/`/proc/<pid>/cmdline` before and
  after, and by a live `list_pages` + navigation round-trip after the correct file
  was patched.


## c26 — 2026-10-17 — browser-terminal copy/paste: why Ctrl+C/V cannot copy, and the tmux recipe

- `webssh`: new **Clipboard** section. Ctrl+C / Ctrl+V never copy or paste in a
  webssh terminal, and the cause is xterm.js rather than tmux or the pane. The
  bundled `xterm.min.js` `_keyDown` calls `triggerDataEvent(t.key)` and then
  `cancel(ev, true)` for every translated key, so `0x03` / `0x16` go to the
  server while the browser's native copy/paste default action is suppressed.
  Recorded the two follow-on facts that make this a dead end rather than a
  mis-setting: xterm.js ships no copy implementation and defaults to
  `rendererType:"canvas"` (no DOM selection, so the context menu offers "Copy
  image"), and it has no OSC 52 support at all (`grep -c 'osc52\|setClipboard'`
  = 0), so `set-clipboard on` cannot reach the browser clipboard. Noted that the
  chords are not inert in a raw-mode TUI either — Claude Code reads `0x03` as
  *cancel* and `0x16` as *paste image*. Documented the client-side
  `attachCustomKeyEventHandler` patch as the only route to the system clipboard,
  with the warning that its paste half is unproven (Chrome blocks
  `execCommand('paste')`, `navigator.clipboard` is undefined off a secure
  context) and that `uv tool upgrade webssh` overwrites the patched assets.
- `base-tools`: new **tmux configuration** section carrying the full
  `~/.tmux.conf` (with an idempotency guard that timestamps a backup of any
  existing config), an apply step via `tmux source-file` that needs no session
  restart, and a verify loop. `mouse on` + `mode-keys vi` move copying into tmux
  copy-mode, where drag / double-click / triple-click all copy on release —
  those three bindings already exist in tmux 3.4, so no `bind` is needed for
  them. `set-clipboard on` additionally wires OSC 52 on native terminals and is
  documented as a no-op in the browser; `history-limit 50000` because agent/TUI
  output outgrows the 2000-line default.
- `base-tools` pitfall: **this machine's tmux 3.4 `copy-mode-vi` table does not
  match the usual documentation.** Measured — `v` is `rectangle-toggle`, and `y`
  has no binding at all, so selecting a line and pressing `y` leaves the session
  in copy-mode with an unchanged buffer. The real keys are `Space` (begin) and
  `Enter` (copy). The shipped config rebinds `v` / `y` to the familiar behaviour,
  keeping rectangular selection on `C-v`; verify with
  `tmux list-keys -T copy-mode-vi | grep -E ' (v|y) '` instead of trusting a
  tutorial. Also documented that `set -g mouse on` is server-wide (scope per
  session with `set -w`) and that key-table changes take effect immediately
  while mouse *negotiation* needs a re-attach (`prefix d`, `tmux a`).
- Correction: a prior note claimed a webssh "keymap addon adds Ctrl+C/CtrlV
  clipboard behavior". No such code existed — `static/` held only the stock
  `xterm.min.js` + `main.js`, `main.js` had no keydown/clipboard handling, and
  neither `catalog/webssh.md` nor the c24 entry mentioned it. The requirement is
  now recorded honestly as an unimplemented client-side patch with its caveats.
- `INDEX.yaml` summaries for both components extended to match. Verified live on
  light-node-01 on 2026-10-17: config loaded into the running server with all
  four sessions (aios / canvas / cyrus / temp) intact, and copy-mode
  copy → paste exercised in a throwaway session (`pane_in_mode` 1 → 0, buffer
  content correct, both before and after the `v`/`y` rebind).
## c27 — 2026-10-17 — new optional component: mmx-cli

- Added `catalog/mmx-cli.md` and registered the `mmx-cli` component in
  `catalog/INDEX.yaml` (group `ai-cli`, channel `npm`, deps `[node-nvm]`).
- Component is intentionally **not** added to any preset (`minimal` /
  `remote-desktop` / `ai-toolkit` / `full`); users opt in explicitly. Rationale:
  mmx is a MiniMax-specific CLI that requires its own auth flow, and most
  vps2desktop deployments don't use MiniMax directly.
- npm package name is **`mmx-cli`** (CLI exposes the `mmx` binary); the same
  `for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done` install
  pattern as codex / opencode / pi / agent-browser, including the npm ≥ 12
  `--allow-scripts=mmx-cli` note.
- Verified on light-node-01 against npm registry at install time: latest
  version is `1.0.28`, dependency tree pulls 33 packages, no native build step,
  binary ships prebuilt. Postinstall was non-blocking on this Node but the
  guard flag is documented for parity with the other Node CLIs.


## c28 — 2026-10-19 — new optional component: browser-mcp (sandboxed browser MCP servers)

- Added `catalog/browser-mcp.md` and registered the `browser-mcp` component in
  `catalog/INDEX.yaml` (group `ai-cli`, channel `npm`, deps
  `[chrome, node-nvm, claude-code]`).
- Component is intentionally **not** added to any preset (`minimal` /
  `remote-desktop` / `ai-toolkit` / `full`), following the `mmx-cli` precedent
  (c27). Rationale: it only matters for Claude Code users, it pulls in `chrome`,
  and it replaces a working-but-unsafe `--no-sandbox` configuration with a
  different privilege model — a security decision the user should opt into.
- **Why**: on a root-managed VPS Chrome refuses to start as root
  (crbug.com/638180). The workaround carried since c25 was
  `--chromeArg=--no-sandbox` for `chrome-devtools-mcp`, which disables the
  browser's kernel-level protections: a hostile page can then reach the
  root-owned host. The component instead starts both MCP servers as a dedicated
  unprivileged account via `setpriv`, so Chrome keeps its real sandbox — the
  kernel provides user namespaces and Chrome's own `chrome_sandbox` is setuid
  root, so no flag is needed at all.
- The run account is `browsermcp`, deliberately **not** `ubuntu`: most Ubuntu
  cloud images ship a vendor `ubuntu` login user, and Rootify (ADR-0003)
  requires leaving that account untouched. It is created passwordless; give it
  a password only for hand-inspection, never in the manifest (ADR-0002).
- Two config locations, verified live: `playwright` is configured by the
  `.mcp.json` next to the plugin manifest (its `plugin.json` carries no embedded
  `mcpServers`), while `chrome-devtools` is configured by the embedded
  `mcpServers` in `.claude-plugin/plugin.json`. The root-level `mcp.json` is the
  agent-plugins.org cross-tool spec that Claude Code ignores — the component
  patches it anyway so switching clients later does not land on a broken config.
- Both plugins auto-upgrade into a fresh version directory holding a clean,
  unpatched config, which silently re-breaks the browser. The component installs
  the `browser-mcp-root-ns` skill with a one-command repair script
  (`bash ~/.claude/skills/browser-mcp-root-ns/scripts/repair-browser-mcp.sh`)
  plus the mandatory `/reload-plugins` step.
- Supporting details the component has to carry: the run account cannot read
  `/root/.npm` (own `npm_config_cache`), `/root/.Xauthority` is 600 root (own
  copy, or the launcher falls back to `--headless`), the Node path under nvm
  must not be hardcoded, and `playwright` resolves explicitly named output
  files against the root-owned workspace (hence `--output-dir`).
- `catalog/chrome.md` now points at this component instead of recommending
  `--no-sandbox` for the agent-driven browser. The desktop wrapper's
  `--no-sandbox` for root *desktop* sessions is a separate case and stays.

## c29 — 2026-10-20 — browser-mcp: let the run account write to the workspace root

- The launchers now grant the run account access to the workspace directory — the
  directory Claude Code was started in — with a POSIX ACL
  (`setfacl -m u:<run user>:rwx <workspace>`). Without it a named screenshot
  failed with `EACCES: permission denied`: the MCP resolves explicitly named
  output files against the workspace root, and `--output-dir` only covers
  automatically named ones.
- Additive and scoped to the one unprivileged account; undo with
  `setfacl -x u:<run user> <dir>`. The launcher skips it when `setfacl` is
  missing (package `acl`, now installed by the component), in which case named
  outputs keep failing and the filename should be omitted instead.
- Found during the first real tool-call pass on light-node-01: navigation,
  snapshot and a full chrome-devtools a11y tree all worked, but
  `browser_take_screenshot` with an explicit filename did not.

## c30 — 2026-10-20 — fixes from the contabo deploy (Ubuntu 26.04)

- xfce-desktop: the Verify line `xfce4-terminal --version` needs an X display —
  on a console-only VPS GTK aborts at init with `cannot open display:` and
  exits 1 *before* printing the version, so Verify produced empty output and
  read as a failed install even though all four packages were installed
  correctly. The Verify now checks the four package states and reads the
  version from `dpkg-query -W -f='${Version}' xfce4`, which is headless-safe
  and is what sibling machines record (ovh: `xfce4 4.20.1`).
- node-nvm: `nvm install --lts` does not export the node bin dir into the
  calling shell, so the Install block's own `npm install -g npm@latest pnpm`
  failed with `npm: command not found` — Node was on disk but npm was not on
  PATH. The block now puts the bin dir on PATH before the npm call, and the
  pitfall spells the trap out for the downstream npm components.
- browser-mcp: Claude Code 2.1.303 no longer ships the `claude-plugins-official`
  marketplace built in. `claude plugin install playwright@claude-plugins-official`
  failed with *"Plugin 'playwright' not found in marketplace
  'claude-plugins-official'"*, and the repair script's cache paths
  (`cache/claude-plugins-official/…`) had nothing to patch. The Install block
  now adds the marketplace from `github.com/anthropics/claude-plugins-official`
  first. The built-in `anthropic-plugin-directory` does carry `playwright`, but
  it installs under a path the repair script does not scan, so a plugin taken
  from there stays unpatched — install from `claude-plugins-official`, with
  `--replace` when the CLI reports the plugin as already present from the other
  listing.
- pi, cyrus: npm ≥ 12 blocks postinstall scripts, and the *transitive*
  packages are warned about separately from the direct one — pi pulls
  `protobufjs` and `@google/genai`, cyrus-ai pulls `cloudflared`. Naming only
  the direct package in `--allow-scripts` leaves those blocked. Both components
  now name every package.
- anaconda: the pitfall's size estimate was low. Measured 2026-10-20 on a
  fresh Ubuntu 26.04: `Anaconda3-2026.07-1-Linux-x86_64.sh` is a 1.14 GB
  download and `/root/anaconda3` ends up at 6.9 GB.
- browser-mcp: the launchers run `npx` from the nvm-managed Node under
  `/root/.nvm`, but a stock `/root` is mode `700` — the unprivileged run account
  got `env: 'npx': Permission denied` and no MCP server could start. The Install
  block now widens `/root` to `711` (traversable, still not listable — the
  arrangement light-node-01 already uses) and makes the Node tree
  world-readable, with a named-user POSIX ACL documented as the narrower
  alternative. The pitfall asks for the check in both directions: the run
  account must execute `node`, and must still fail to `ls /root`.

## c31 — 2026-10-20 — chatgpt: the sign-in can be verified afterwards

- The "Sign in to ChatGPT in the application" follow-up had no way to be
  checked, so a finished sign-in was indistinguishable from a pending one. The
  component doc now documents the read-only check `codex login status`:
  `Logged in using ChatGPT` confirms a live account session. The check reads
  state only — it never drives a login flow and never prints token values.
- The desktop app is an Owl/Electron build whose Chromium profile is
  `~/.config/Codex` (`UserDataDirectoryName=Codex` in `resources/owl-app.ini`),
  not `~/.config/chatgpt`. Searching for a `chatgpt` profile directory finds
  nothing and can be misread as "the app never ran". The shared account session
  is in `~/.codex/auth.json` (`auth_mode: chatgpt`), which is separate from
  the Codex CLI's own `~/.codex` home.
- The id token inside `~/.codex/auth.json` expires within hours while the
  session is carried by the refresh token, so a stale id token is not a
  logged-out app. Judge the sign-in by `codex login status`.

## c32 — 2026-10-21 — chatgpt: the vendor publishes the Linux package itself

- The component doc said the DEB had to be supplied by the user, which left no
  way to install the app without hunting for the file first. OpenAI ships the
  Linux build itself, and the upstream documentation
  (https://developers.openai.com/codex/linux/linux-app) publishes the exact
  package URL for every supported distribution and architecture — for
  Ubuntu/Debian on x64:
  https://persistent.oaistatic.com/codex-app-prod/linux/deb/latest/chatgpt_amd64.deb
  The doc now lists the supported matrix (Ubuntu 24.04/26.04 LTS, Debian 13,
  Fedora 43/44, Arch Linux; x64 and ARM64) and the URLs. A user-supplied DEB
  still takes precedence; the official URL is the fallback, not a substitution.
- The package's maintainer scripts register a signed OpenAI APT repository, so
  later versions arrive through `apt-get update && apt-get install
  --only-upgrade chatgpt` instead of a fresh download. Run `apt-get update`
  after the install — until then the vendor repository is not in the package
  lists and only the locally installed version is visible to apt.
- Verified live on a fresh Ubuntu 26.04.1 install: package `chatgpt`
  26.1016.51333 (amd64, 527 MB), root-safe wrapper plus trusted desktop and
  menu launchers, and one visible GUI window belonging to
  `/usr/lib/chatgpt/ChatGPT` on the Xfce session.

## c33 — 2026-10-21 — cyrus: install from source instead of the npm global package

- The `cyrus` Component now checks out `github.com/cyrusagents/cyrus`, builds it
  with `pnpm install && pnpm build`, and links `apps/cli` with `npm link`. The
  npm-global `cyrus-ai@latest` install is gone; the Install block removes a
  pre-existing global package so the command cannot silently keep running the
  old code.
- Rationale: a source install lets a deployment run and keep tested changes
  that are not on npm yet — at the time of writing the repository was at 0.2.74
  while npm `latest` was still 0.2.73.
- Guard and Verify now assert the resolved path of `cyrus`, not only its
  version — an npm package and a source build both print a version number.
- New "Updating a source install" section: `git pull --ff-only`,
  `pnpm install`, `pnpm build`, then restart the worker; a running process keeps
  the code it loaded at startup.
- Known Pitfalls: replaced the npm ≥12 postinstall note with pnpm 10's blocked
  dependency build scripts (`cloudflared` is expected to be ignored because the
  tunnel is external); documented that `pnpm link -g` cannot link this monorepo
  (`ERR_PNPM_NO_GLOBAL_BIN_DIR` on a stock box, then
  `ERR_PNPM_WORKSPACE_PKG_NOT_FOUND` on `workspace:*` deps) and that `npm link`
  is the supported path; added the checkout-path coupling warning.

## c34 — 2026-10-21 — fixes from a live contabo deploy (Ubuntu 26.04.1)

- bash-completion: the Verify greped for `_completion_loader`, but
  bash-completion 2.16 (the 26.04 package) registers the default handler as
  `_comp_complete_load`. The install was working — the assertion was written
  against an older package. The Verify now accepts either name, and the
  expected script count is documented for both 24.04 (936) and 26.04 (1378).
- notepadplusplus: the URL extraction used `grep -oE` with `\K`, which is a
  PCRE construct that GNU grep only honours under `-P`. The pattern matched
  nothing, `DEB_URL` came out empty and `curl` died with "blank argument".
  Fixed to `-oP` — the only other catalog doc that used `\K` already did.
- notepadplusplus: the release deb's `Depends` names the 24.04 Qt6 packages
  (`libqt6widgets6t64`), which no longer exist in 26.04, so `apt-get install
  ./npp.deb` fails with "not installable". The doc now documents repacking
  the deb with the 26.04 names (`libqt6widgets6`, `libqt6xml6`) instead of
  forcing the install. `libqt6core6t64` still exists and needs no rewrite.
- docker-desktop: documented what a host without `/dev/kvm` actually does.
  The `test -c /dev/kvm` precondition aborts the whole Install script on a
  cloud guest with no nested virtualization, while the package itself
  installs and the Verify passes — only the runtime is dead. The doc now
  says to skip the component or record it as installed-but-unusable.
- docker-desktop: the package installs its own CLI plugins into
  `/usr/lib/docker/cli-plugins/`, which take precedence over the Engine's
  `/usr/libexec/docker/cli-plugins/`. With both installed, `docker compose
  version` reports Desktop's bundled Compose (v5.6.0) instead of the
  `docker-compose-plugin` package version (v5.6.1). The Engine daemon,
  socket and `docker run` are unaffected.

All findings verified live on contabo (Ubuntu 26.04.1, x86_64) on 2026-10-21.

## c35 — 2026-10-22 — tailscale tray verification gotchas (contabo)

- tailscale: `com.canonical.dbusmenu.GetLayout` with `recursionDepth 0`
  returns an **empty** layout — the systray fills the child list only for a
  recursive layout. Verifying the menu with depth `0`, the natural first
  guess, makes a working menu look broken. Use `int32:-1`.
- tailscale: `busctl` cannot pass `-1` as an argument (it parses it as an
  option: `invalid option -- '1'`) and rejects the empty `as` argument in its
  own array syntax. The doc now verifies the menu with `dbus-send`.
- tailscale: the `IconPixmap` property is **ARGB in network byte order** —
  the first byte of each pixel is the alpha channel. Decoding it as BGRA,
  the usual Linux memory layout, yields a wrong-coloured icon with a broken
  alpha channel.
- tailscale: the status icon is a **separate process**, not part of the
  daemon. A correctly installed and joined `tailscaled` still shows no tray
  icon until `tailscale systray` runs inside a desktop session; the CLI
  install alone never provides an icon. The doc now states this and notes
  that the tray exists only while a session is logged in.
- tailscale: the "Desktop integration (optional)" section now ships the
  exact verification commands and defines the compatibility evidence — an
  active `tailscale` StatusNotifierItem plus a non-empty recursive menu
  layout — before the autostart entry may be persisted.

All findings verified live on contabo (Ubuntu 26.04.1, x86_64) on 2026-10-22.
