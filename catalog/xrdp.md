# xrdp remote desktop

| | |
|---|---|
| id | `xrdp` |
| group | desktop |
| deps | `xfce-desktop` |
| channel | apt |

xrdp + xorgxrdp with TLS hardened and the session pinned to Xfce. This is the component that turns the box into a GUI-reachable machine.

## Guard (idempotency)

```bash
systemctl is-active xrdp 2>/dev/null | grep -q active && ss -ltn | grep -q ':3389 ' && echo present
```

## Install

```bash
export DEBIAN_FRONTEND=noninteractive
apt-get install -y xrdp xorgxrdp ssl-cert
adduser xrdp ssl-cert   # idempotent: fails harmlessly if already a member

# TLS: negotiate a modern layer, high crypt level, strong ciphers (TLS 1.2+)
sed -i 's/^#\?security_layer=.*/security_layer=negotiate/; s/^#\?crypt_level=.*/crypt_level=high/; s/^#\?tls_ciphers=.*/tls_ciphers=HIGH/' /etc/xrdp/xrdp.ini

# Session: Xfce for every user that logs in via RDP
grep -qxf xfce4-session /root/.xsession 2>/dev/null || echo xfce4-session > /root/.xsession
chmod +x /root/.xsession

systemctl enable --now xrdp
```

## Verify

```bash
systemctl is-active xrdp && ss -ltn | grep ':3389 ' && grep -E '^(security_layer|crypt_level)' /etc/xrdp/xrdp.ini
```

## Known Pitfalls

- **The RDP password is the system password** — after Rootify that is the root password (same one the user gave the agent).
- In the RDP client, session type "Xorg"; leave color depth default on first try.
- xrdp 0.9.x does not export `XRDP_SESSION` into sessions (0.10.x does) — the `xrdp-audio` component carries a workaround.
- Restarting xrdp disconnects live RDP sessions — schedule it when nobody is attached.
