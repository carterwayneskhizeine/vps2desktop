# Base toolset

| | |
|---|---|
| id | `base-tools` |
| group | base |
| deps | none |
| channel | apt |

The foundation package set every machine should have, so later components and daily work never hit "command not found".

## Guard (idempotency)

```bash
dpkg -s git jq ripgrep tmux htop gh >/dev/null 2>&1 && echo present
```

## Install

```bash
export DEBIAN_FRONTEND=noninteractive
apt-get install -y \
  ca-certificates curl wget git gnupg unzip zip xz-utils bzip2 \
  build-essential pkg-config \
  python3 python3-venv python3-pip \
  jq ripgrep tmux htop rsync \
  gh sqlite3 xclip xsel
```

## Verify

```bash
git --version && jq --version && rg --version | head -1 && gh --version | head -1 && python3 --version
```

## tmux configuration (copy/paste in browser terminals)

Any browser-based SSH client swallows <kbd>Ctrl</kbd>+<kbd>C</kbd> and
<kbd>Ctrl</kbd>+<kbd>V</kbd>: webssh's xterm.js `preventDefault()`s both and ships
`0x03` / `0x16` to the remote instead (see the `webssh` Clipboard section). Agent
and TUI output is then uncopyable, which hurts on a headless VPS where you are
almost always working through exactly such a terminal. The config below moves
copying into tmux copy-mode, where it works everywhere. It is harmless on a
native terminal — there it additionally wires OSC 52 so the system clipboard
follows the tmux buffer.

Drop this `~/.tmux.conf` on every machine whose terminal is browser-based; it is
the same file on all of them.

### Guard (idempotency)

Keep whatever was there before — do not clobber a hand-tuned config:

```bash
[ -e /root/.tmux.conf ] && cp -a /root/.tmux.conf /root/.tmux.conf.bak.$(date +%F-%H%M%S)
```

### Install

```bash
install -m 0644 /dev/stdin /root/.tmux.conf <<'TMUXCONF'
# tmux configuration — copy/paste inside browser terminals
#
# Why (measured 2026-10-17, light-node-01, webssh 1.6.3 / xterm.js):
#   Browser -> webssh -> ssh -> tmux -> Claude Code
#   xterm.js's _keyDown preventDefault()s every key it translates, so Ctrl+C
#   leaves as 0x03 and Ctrl+V as 0x16 while the browser's own copy/paste is
#   suppressed. Claude Code runs the tty in raw mode (ISIG off) and reads those
#   two bytes as "cancel" and "paste image / verbatim".
#   => Ctrl+C / Ctrl+V can never copy or paste in a browser terminal.
#      Copying has to go through tmux copy-mode.

# Hand the mouse to tmux: drag-select then release copies. tmux 3.4 already
# binds MouseDrag1Pane -> begin-selection and MouseDragEnd1Pane /
# DoubleClick1Pane / TripleClick1Pane -> copy-pipe-and-cancel, so no extra
# bind is needed for drag, double-click (word) or triple-click (line).
set -g mouse on

# vi keys in copy-mode.
set -g mode-keys vi

# tmux 3.4's copy-mode-vi table does NOT match the common documentation
# (measured on this machine): `v` is rectangle-toggle and `y` is not bound at
# all; the real "start selection" key is Space and the real "copy" key is
# Enter. Put the familiar keys back. Rectangular selection stays on C-v.
bind -T copy-mode-vi v send-keys -X begin-selection
bind -T copy-mode-vi y send-keys -X copy-pipe-and-cancel

# Stop ESC from waiting 500ms so the prefix key and Alt chords respond at once.
set -g escape-time 0

# Also emit OSC 52 on copy so a native terminal's clipboard follows the tmux
# buffer. webssh's xterm.js has no OSC 52 support at all (measured: grep hits
# 0), so this is a no-op in the browser — copying stops at the tmux buffer.
set -g set-clipboard on

# Agent/TUI output easily exceeds the 2000-line default.
set -g history-limit 50000

# Usage
#   drag / double-click / triple-click -> copy on release
#   prefix + [   copy-mode: v start -> move -> y copy and leave
#   prefix + ]   paste the tmux buffer; press it twice to ask a native terminal
#                client to paste the system clipboard
#   prefix + d ; tmux a   re-attach after changing anything mouse-related, so
#                the client re-negotiates mouse reporting
#
# Other copy-mode keys on this tmux
#   Space / v   start selection       Enter / y   copy and leave
#   V           select the whole line q / C-c     leave without copying
#   Escape      clear the selection    g / G       jump to top/bottom of history
#   arrows hjkl move the cursor and grow the selection; the wheel works too
#   / ? n       search forward / backward / next match
#   :N Enter    jump to line N
#   C-v         rectangular (block) selection
TMUXCONF

# Apply to the running server — every session picks it up, nothing restarts.
tmux source-file /root/.tmux.conf 2>/dev/null || true
```

### Verify

```bash
for o in mouse mode-keys escape-time set-clipboard history-limit; do
  printf '%-14s %s\n' "$o" "$(tmux show-options -g "$o" 2>&1)"
done

# v must start a selection and y must copy-and-exit, not the other way round
tmux list-keys -T copy-mode-vi | grep -E ' (v|y) '
```

Expected: `mouse on`, `mode-keys vi`, `escape-time 0`, `set-clipboard on`,
`history-limit 50000`, and `v → begin-selection`, `y → copy-pipe-and-cancel`.

`source-file` re-reads the file into the running server without restarting
sessions, and key-table changes take effect on the next copy-mode entry. Only
*terminal negotiation* — mouse reporting above all — needs a re-attach
(`prefix d`, then `tmux a`).

## Known Pitfalls

- `gh` needs `gh auth login` before use — manual, never automated.
- **This machine's tmux 3.4 `copy-mode-vi` table does not match the usual
  documentation.** Measured with `tmux list-keys -T copy-mode-vi`: `v` is
  `rectangle-toggle`, and `y` has **no binding at all** — selecting a line and
  pressing `y` leaves you sitting in copy-mode with an unchanged buffer. The real
  keys are `Space` to start a selection and `Enter` to copy. The config above
  rebinds `v` / `y` to the familiar behaviour (rectangular selection remains on
  `C-v`); check with `tmux list-keys -T copy-mode-vi | grep -E ' (v|y) '` rather
  than trusting a tutorial, and never assume a key table is the one you remember.
- **`set -g mouse on` is server-wide.** It applies to every session on that tmux
  server, including ones running other agents. Scope it down per session with
  `set -w mouse off` if a session needs the mouse for its own UI.
- **`set-clipboard on` is a no-op in browser terminals.** OSC 52 is unsupported
  by webssh's xterm.js, so a tmux copy lands in the tmux buffer only — there is
  no way to get it into the *system* clipboard from the server side.

## Manual follow-ups

- none
