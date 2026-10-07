# Chinese input (fcitx5)

| | |
|---|---|
| id | `fcitx5-chinese` |
| group | desktop |
| deps | `xfce-desktop` |
| channel | apt |

fcitx5 with the Chinese addons (pinyin + cloud pinyin) and system-wide IM environment variables.

## Guard (idempotency)

```bash
dpkg -s fcitx5 fcitx5-chinese-addons >/dev/null 2>&1 && grep -q XMODIFIERS /etc/environment && echo present
```

## Install

```bash
export DEBIAN_FRONTEND=noninteractive
apt-get install -y fcitx5 fcitx5-chinese-addons fcitx5-frontend-all

# System-wide IM environment (guard each line)
for line in 'GTK_IM_MODULE=fcitx' 'QT_IM_MODULE=fcitx' 'XMODIFIERS=@im=fcitx'; do
  grep -qxF "$line" /etc/environment || echo "$line" >> /etc/environment
done

# Default input method for the session (write manually — im-config is flaky over SSH)
echo 'run_im fcitx5' > /root/.xinputrc
```

## Verify

```bash
dpkg -s fcitx5-chinese-addons | grep -q 'Status: install ok installed' && grep -c '_IM_MODULE\|XMODIFIERS' /etc/environment
```

## Known Pitfalls

- **Logout/login required** — inside the RDP session, log out fully and back in; reconnecting a saved session is not enough (a classic trap).
- Toggle with `Ctrl+Space` once the tray icon shows fcitx5.
- `im-config -n fcitx5` sometimes does not persist over SSH — that is why `.xinputrc` is written directly.
- If pinyin candidates show as boxes, `fonts-noto-cjk` (from `xfce-desktop`) is missing.
- If input switching works but the panel icon is missing, first ensure the
  `systray` panel plugin using the live-session procedure in `xfce-desktop.md`.
  Modern Xfce includes StatusNotifier support in that plugin; the notification
  plugin alone does not display Fcitx5's icon. Verify registration with the
  session's `org.kde.StatusNotifierWatcher` before restarting Fcitx5 or changing
  the working input-method configuration.
