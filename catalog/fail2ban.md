# fail2ban (sshd jail)

| | |
|---|---|
| id | `fail2ban` |
| group | base |
| deps | none |
| channel | apt |

Bans source IPs that repeatedly fail SSH auth. Strongly recommended: after Rootify, root password login is exposed to the public internet and scanners will hammer it all day.

## Guard (idempotency)

```bash
systemctl is-active fail2ban 2>/dev/null | grep -q active && fail2ban-client status sshd >/dev/null 2>&1 && echo present
```

## Install

```bash
export DEBIAN_FRONTEND=noninteractive
apt-get install -y fail2ban
systemctl enable --now fail2ban
# On Debian/Ubuntu the sshd jail is enabled by default via defaults-debian.conf.
```

## Verify

```bash
systemctl is-active fail2ban && fail2ban-client status sshd | head -3
```

## Known Pitfalls

- **The agent itself can get banned**: opening many *new* SSH connections in a short window can trip the jail even with successful auth. This is why SKILL.md mandates one multiplexed ControlMaster connection.
- Unban yourself if it happens: `fail2ban-client set sshd unbanip <ip>` (needs an already-open session or provider console/VNC).
- Ban windows default to 10 minutes (bantime), then auto-expire.
