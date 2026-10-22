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
#
# The guards below test root's OWN runtime-dir sockets, never a bare process
# name: `pgrep -x pipewire` matches every user's daemon, and the lightdm
# greeter session always has one, so root's instance was never started.
mkdir -p /root/.local/bin /root/.config/autostart
cat > /root/.local/bin/xrdp-audio-root.sh <<'EOF'
#!/bin/bash
# Autostart PipeWire for root xrdp sessions (ConditionUser=!root workaround).
#
# Ubuntu's pipewire user units carry ConditionUser=!root, so they never start
# for root. This script starts root's own PipeWire stack from inside the xrdp
# session (where xrdp exports XRDP_SESSION / XRDP_SOCKET_PATH) and loads the
# xrdp channel module so the session gets a real "xrdp-sink" instead of a
# silent Dummy Output.
#
# Fixed 2026-10-22: the guard used a bare `pgrep -x pipewire`, which matches
# ANY user's daemon. The lightdm greeter session always has one, so root's
# own instance was never started and load_pw_modules.sh aborted with
# "pipewire server is not running". Guard on root's own runtime-dir socket
# and on per-user process ids instead of the bare process name.
LOG=/root/.xrdp-audio-root.log
exec >>"$LOG" 2>&1
echo "===== $(date "+%F %T") XRDP_SESSION=${XRDP_SESSION:-?} socket=${XRDP_SOCKET_PATH:-?} DISPLAY=${DISPLAY:-?} ====="

export XDG_RUNTIME_DIR="${XDG_RUNTIME_DIR:-/run/user/0}"
export XRDP_SESSION="${XRDP_SESSION:-1}"
export XRDP_SOCKET_PATH="${XRDP_SOCKET_PATH:-/run/xrdp/sockdir/$(id -u)}"

# xrdp's channel server publishes the audio sockets as
# xrdp_chansrv_audio_{out,in}_socket_<display-number>. xrdp >= 0.10 exports
# them into the session environment; derive them from DISPLAY when missing.
DISPLAY_NUM="${DISPLAY#:}"; DISPLAY_NUM="${DISPLAY_NUM%%.*}"
export XRDP_PULSE_SINK_SOCKET="${XRDP_PULSE_SINK_SOCKET:-xrdp_chansrv_audio_out_socket_${DISPLAY_NUM}}"
export XRDP_PULSE_SOURCE_SOCKET="${XRDP_PULSE_SOURCE_SOCKET:-xrdp_chansrv_audio_in_socket_${DISPLAY_NUM}}"

mkdir -p "$XDG_RUNTIME_DIR" "$XDG_RUNTIME_DIR/pulse" 2>/dev/null

# Wait for the channel server to publish its audio sockets; without them the
# module loads but silently produces no sink.
for i in $(seq 1 30); do
  [ -S "$XRDP_SOCKET_PATH/$XRDP_PULSE_SINK_SOCKET" ] && break
  sleep 0.5
done

# Start root's own PipeWire stack. The module is instantiated inside the
# daemon, so the daemon must inherit XRDP_SOCKET_PATH and the two pulse
# socket names from above.
if [ ! -S "$XDG_RUNTIME_DIR/pipewire-0" ]; then
  setsid nohup pipewire >>"$LOG" 2>&1 </dev/null &
fi
for i in $(seq 1 40); do
  [ -S "$XDG_RUNTIME_DIR/pipewire-0" ] && break
  sleep 0.5
done

if [ ! -S "$XDG_RUNTIME_DIR/pulse/native" ]; then
  setsid nohup pipewire-pulse >>"$LOG" 2>&1 </dev/null &
fi
if ! pgrep -u "$(id -u)" -x wireplumber >/dev/null 2>&1; then
  setsid nohup wireplumber >>"$LOG" 2>&1 </dev/null &
fi

for i in $(seq 1 30); do
  timeout 3 pw-cli ls Node >/dev/null 2>&1 && break
  sleep 0.5
done
/usr/libexec/pipewire-module-xrdp/load_pw_modules.sh
sleep 1
wpctl status 2>/dev/null | sed -n "/Sinks:/,/Sources:/p"
pactl list short sinks 2>/dev/null | sed 's/^/[sink] /'
pactl info 2>/dev/null | grep -i 'default sink' | sed 's/^/[info] /'
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
pactl info | grep 'Default Sink'   # must be "xrdp-sink", not "Dummy Output"
pactl list short sinks              # must list an "xrdp-sink" entry
```

Then play any audio (e.g. a YouTube video in Chrome). A headless server has no
sound card, so a `(null)` sink is *also* listed — that is normal. The check is
that `Default Sink` is `xrdp-sink`.

To test the path without a media file:

```bash
python3 -c "import wave,struct,math; w=wave.open('/tmp/tone.wav','w'); w.setnchannels(2); w.setsampwidth(2); w.setframerate(44100); w.writeframes(b''.join(struct.pack('<hh',int(12000*math.sin(2*math.pi*440*i/44100)),int(12000*math.sin(2*math.pi*440*i/44100))) for i in range(44100*3))); w.close()"
paplay --device=xrdp-sink /tmp/tone.wav
```

## Known Pitfalls

- **Logout/login, not reconnect**: audio starts on a fresh session start; reconnecting to a saved session keeps the old (silent) state.
- **A bare `pgrep -x pipewire` guard breaks root sessions.** It matches *every*
  user's daemon. On a box with a lightdm greeter, root's own PipeWire is then
  never started, `load_pw_modules.sh` aborts with `pipewire server is not
  running, exiting`, and the session has no PulseAudio server at all
  (`pactl` → `Connection refused`). Guard on `$XDG_RUNTIME_DIR/pipewire-0`
  and `$XDG_RUNTIME_DIR/pulse/native` instead, and use `pgrep -u "$(id -u)"`
  when a process check is needed.
- **The xrdp module is instantiated inside the PipeWire daemon**, so it reads
  `XRDP_SOCKET_PATH`, `XRDP_PULSE_SINK_SOCKET` and `XRDP_PULSE_SOURCE_SOCKET`
  from the *daemon's* environment — not from the `pw-cli` that loads it.
  Export them before starting `pipewire`, or the module loads but creates no
  sink.
- The module needs `xrdp-chansrv` to have published
  `xrdp_chansrv_audio_out_socket_<display>` in `$XRDP_SOCKET_PATH` first; wait
  for the socket instead of loading the module blindly.
- Sink stuck on "Dummy Output" → check `/root/.xrdp-audio-root.log`. On xrdp
  0.9.x it is the missing `XRDP_SESSION`/`XRDP_SOCKET_PATH` exports; on xrdp
  0.10+ those are exported already, so look for a *different user's* pipewire
  instance (see the `pgrep` pitfall above).
- `pw-metadata` prints `Failure: Not supported` for the clock settings on
  pipewire 1.6 — harmless, the sink is still created.
- On distributions where `pipewire-module-xrdp` is not packaged, audio cannot be provided — the agent should report this instead of improvising.
