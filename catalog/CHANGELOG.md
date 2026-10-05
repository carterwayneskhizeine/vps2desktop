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
