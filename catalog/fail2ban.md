# fail2ban (sshd jail)

| | |
|---|---|
| id | `fail2ban` |
| group | base |
| deps | none |
| channel | apt |

Bans source IPs that repeatedly fail SSH auth. Strongly recommended: after
Rootify, root password login is exposed to the public internet and scanners
will hammer it all day.

This component installs the service and the distro's stock `sshd` jail, which
only catches bursts. Slow scanners, sub-threshold bursts and repeat offenders
need the `sshd-tiers` component (deps: this one).

## Guard (idempotency)

```bash
systemctl is-active fail2ban 2>/dev/null | grep -q active \
  && fail2ban-client status sshd >/dev/null 2>&1 && echo present
```

## Install

```bash
export DEBIAN_FRONTEND=noninteractive
apt-get install -y fail2ban
systemctl enable --now fail2ban
# On Debian/Ubuntu the sshd jail is enabled by default via defaults-debian.conf,
# which also selects the ban action (nftables on current releases).
```

## Verify

```bash
systemctl is-active fail2ban && fail2ban-client status sshd | head -3
fail2ban-client -t                                   # config parses
nft list chain inet f2b-table f2b-chain >/dev/null && echo nft-ok
```

## Known Pitfalls

- **The agent itself can get banned**: opening many *new* SSH connections in a
  short window can trip the jail even with successful auth. This is why
  SKILL.md mandates one multiplexed ControlMaster connection.
- Unban yourself if it happens: `fail2ban-client set sshd unbanip <ip>` (needs
  an already-open session or provider console/VNC). A manual `banip` only lasts
  for the jail's `bantime` (1 h here); send repeat offenders to `recidive` for
  a real block.
- **Stock bantime is 10 minutes** (from `jail.conf`). Without
  `bantime.increment` a scanner simply returns after every unban, so a flat
  bantime only schedules the comeback. `sshd-tiers` raises the base ban and
  enables exponential escalation.
- **Bans are not in `iptables`** on current releases: Debian's
  `defaults-debian.conf` selects `nftables`, so `iptables -S` shows nothing
  and `nft list chain inet f2b-table f2b-chain` is the real view. On older
  packages the action is `iptables-multiport` — trust
  `fail2ban-client status <jail>`, then inspect whichever backend shows the set.
- **`fail2ban-client get <jail> action` fails** with
  `IndexError: list index out of range` on 1.1.0, even for stock jails. It is a
  CLI parsing bug, not a broken action and not a service fault. Use
  `get <jail> bantime|maxretry|findtime|ignoreip` and `status <jail>`.
- **`/var/log/fail2ban.log` is not the journal.** fail2ban writes its `Ban`
  lines to the file log; the systemd unit's journal holds only a few
  start/stop lines. A jail that reads the journal (e.g. `recidive` with
  `backend = systemd`) matches nothing and stays silent — see `sshd-tiers`.
- **One attempt, several jails.** `sshd`, `sshd-slow` and `sshd-daily` read
  the same sshd log, so a single failed login is counted once per jail; dedupe
  by (IP, timestamp) before reporting totals.
