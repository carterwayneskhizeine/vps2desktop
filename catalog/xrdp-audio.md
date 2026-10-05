# RDP audio (PipeWire)

| | |
|---|---|
| id | `xrdp-audio` |
| group | desktop |
| deps | `xrdp` |
| channel | apt |

Working sound inside the RDP session via `pipewire-module-xrdp`, including the root-session autostart workaround.

## Guard (idempotency)

```bash
dpkg -s pipewire-module-xrdp >/dev/null 2>&1 && test -x /root/.local/bin/xrdp-audio-root.sh && echo present
```

## Install

```bash
export DEBIAN_FRONTEND=noninteractive
apt-get install -y pipewire pipewire-bin pipewire-pulse wireplumber pipewire-module-xrdp pulseaudio-utils

# Root-session workaround: Ubuntu's pipewire user units carry ConditionUser=!root
# (they never start for root), and xrdp 0.9.x does not export XRDP_SESSION /
# XRDP_SOCKET_PATH (0.10+ does). Without this, load_pw_modules.sh silently skips
# and the sink stays "Dummy Output".
mkdir -p /root/.local/bin /root/.config/autostart
cat > /root/.local/bin/xrdp-audio-root.sh <<'EOF'
#!/bin/bash
# Autostart PipeWire for root xrdp sessions (ConditionUser=!root workaround).
LOG=/root/.xrdp-audio-root.log
exec >>"$LOG" 2>&1
echo "===== $(date "+%F %T") XRDP_SESSION=${XRDP_SESSION:-?} socket=${XRDP_SOCKET_PATH:-?} ====="
export XDG_RUNTIME_DIR="${XDG_RUNTIME_DIR:-/run/user/0}"
export XRDP_SESSION="${XRDP_SESSION:-1}"
export XRDP_SOCKET_PATH="${XRDP_SOCKET_PATH:-/run/xrdp/sockdir/$(id -u)}"
mkdir -p "$XDG_RUNTIME_DIR" 2>/dev/null
pgrep -x pipewire       >/dev/null 2>&1 || pipewire &
pgrep -x pipewire-pulse >/dev/null 2>&1 || pipewire-pulse &
pgrep -x wireplumber    >/dev/null 2>&1 || wireplumber &
for i in $(seq 1 30); do timeout 3 pw-cli ls Node >/dev/null 2>&1 && break; sleep 0.5; done
/usr/libexec/pipewire-module-xrdp/load_pw_modules.sh
sleep 1
wpctl status 2>/dev/null | sed -n "/Sinks:/,/Sources:/p"
EOF
chmod 755 /root/.local/bin/xrdp-audio-root.sh

cat > /root/.config/autostart/xrdp-audio-root.desktop <<'EOF'
[Desktop Entry]
Type=Application
Name=xrdp-audio-root
Comment=Start PipeWire manually for root xrdp session (ConditionUser=!root)
Exec=/root/.local/bin/xrdp-audio-root.sh
Terminal=false
X-GNOME-Autostart-enabled=true
EOF
```

## Verify (install-time)

```bash
ls /usr/libexec/pipewire-module-xrdp/load_pw_modules.sh && dpkg -s pipewire-module-xrdp | grep -q 'Status: install ok installed'
```

## Verify (in-session — manual follow-up)

After logging into the RDP session (and one logout/login if it was already open):

```bash
pactl info | grep 'Default Sink'   # must NOT be "Dummy Output"; expect an xrdp sink
```

Then play any audio (e.g. a YouTube video in Chrome).

## Known Pitfalls

- **Logout/login, not reconnect**: audio starts on a fresh session start; reconnecting to a saved session keeps the old (silent) state.
- Sink stuck on "Dummy Output" → check `/root/.xrdp-audio-root.log`; almost always the missing `XRDP_SESSION`/`XRDP_SOCKET_PATH` exports (xrdp 0.9.x) that the script above provides.
- On distributions where `pipewire-module-xrdp` is not packaged, audio cannot be provided — the agent should report this instead of improvising.
