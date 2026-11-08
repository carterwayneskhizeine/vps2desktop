# rdp-guard (fail2ban xrdp jail + kernel guards)

| | |
|---|---|
| id | `rdp-guard` |
| group | desktop |
| deps | [xrdp, fail2ban] |
| channel | apt |

Hardens a public-facing 3389 against RDP scanners and connection floods:
a fail2ban jail that rate-bans by *accepted-connection* frequency, plus two
kernel-level INPUT guards (per-IP concurrent cap, per-IP new-connection rate).
A public RDP port is scanned continuously; without this, one scanner IP can
hold hundreds of concurrent connections and fork one xrdp process each.

## Guard (idempotency)

```bash
fail2ban-client status xrdp >/dev/null 2>&1 \
  && iptables -C INPUT -p tcp --dport 3389 -m connlimit --connlimit-above 10 --connlimit-mask 32 -j DROP 2>/dev/null \
  && echo present
```

## Install

Every accepted TCP connection is logged by xrdp with the peer IP
(`/var/log/xrdp.log`: `Socket 14: connection accepted from [::ffff:1.2.3.4]:p`),
so the jail can ban on connection *rate* — the only signal that catches
clients that never attempt auth.

1. Filter — `/etc/fail2ban/filter.d/xrdp.conf`:

   ```ini
   [Definition]
   failregex = ^\S+ \[\w+\s*\] Socket \d+: connection accepted from \[::ffff:<HOST>\]:\d+$
   datepattern = ^\[{DATE}
                 {^LN-BEG}
   journalmatch =
   ```

2. Jail — `/etc/fail2ban/jail.d/xrdp.conf`. Replace `<owner-public-ip>` with
   the owner's real client IP(s) and keep the actual list only in the
   machine's Local State notes, never in the catalog:

   ```ini
   [xrdp]
   enabled   = true
   port      = 3389
   protocol  = tcp
   backend   = auto
   logpath   = /var/log/xrdp.log
   filter    = xrdp
   # Ban anyone opening more than 6 connections within 10 minutes (>=0.6/min).
   # Normal clients open 1-2 connections per session; slow scanners sustain
   # ~0.7/min while reusing ONE source port (see Pitfalls).
   maxretry  = 6
   findtime  = 600
   bantime   = 1h
   bantime.increment = true
   ignoreip  = 127.0.0.1/8 ::1 100.64.0.0/10 <owner-public-ip>
   ```

   `backend = auto` is required when fail2ban's global default is `systemd`
   (Debian's `defaults-debian.conf`): xrdp logs to a file, not the journal.

3. Kernel guards (belt under the jail) and persistence:

   ```bash
   # Cap per-IP concurrent connections to 3389 — stops flood-style pileups.
   iptables -I INPUT 2 -p tcp --dport 3389 -m connlimit --connlimit-above 10 --connlimit-mask 32 -j DROP
   # Cap per-IP new-connection rate to 3389.
   iptables -I INPUT 3 -p tcp --dport 3389 -m conntrack --ctstate NEW -m recent --name RDPCONN --update --seconds 60 --hitcount 15 -j DROP
   iptables -I INPUT 4 -p tcp --dport 3389 -m conntrack --ctstate NEW -m recent --name RDPCONN --set

   export DEBIAN_FRONTEND=noninteractive
   apt-get install -y iptables-persistent   # debconf saves current rules
   netfilter-persistent save                 # rules survive reboot
   ```

4. `fail2ban-client reload`

## Verify

```bash
fail2ban-client status xrdp | head -4                                  # jail active, counting
iptables -C INPUT -p tcp --dport 3389 -m connlimit --connlimit-above 10 --connlimit-mask 32 -j DROP && echo connlimit-ok
grep -c 3389 /etc/iptables/rules.v4                                    # >= 3 persisted rules
```

Optional sanity: `fail2ban-regex /var/log/xrdp.log /etc/fail2ban/filter.d/xrdp.conf`
should match every `connection accepted` line (one failregex, not two — two
overlapping regexes double-count failures and halve the effective maxretry).

## Known Pitfalls

- **Single-source-port scanners blind the kernel guards**: a client that
  reuses the same source port for every connection shows up as *one*
  conntrack entry, so `connlimit`/`recent` never fire no matter how many
  sockets it holds. Only the fail2ban rate threshold (6/10 min) catches it;
  do not "fix" the threshold upward just because the kernel rules look
  impressive.
- **xrdp never times out silent clients**: every accepted connection forks an
  xrdp process that blocks forever if the peer connects and says nothing.
  connlimit bounds the pile per IP; to reclaim leaked processes, kill them by
  peer (`ss -tnp ... | grep <ip> | pid=`, then `kill -9`). Zombies from other
  parents are diagnosed with `ps -eo ppid,stat,comm | awk '$2 ~ /^Z/'`.
- **Scanners return within ~1 minute of every unban** — `bantime.increment`
  (1h → 2h → 4h → … capped at 1 week) is what actually neutralizes repeat
  offenders; a flat bantime just schedules their comeback.
- **Self-ban risk**: an RDP client in reconnect-storm mode can trip 6/10min.
  Whitelist the owner's IPs in `ignoreip` (tailnet `100.64.0.0/10` is always
  whitelisted), and keep the unban command in the back pocket:
  `fail2ban-client set xrdp unbanip <ip>`.
- **The kernel rules and any manual `iptables -D/I` blocks are NOT persisted
  by fail2ban** — fail2ban bans live in a separate nft table. After changing
  INPUT rules, re-run `netfilter-persistent save` or they vanish on reboot.
- The log's date format is ISO8601 with milliseconds and offset; the filter's
  `datepattern` handles it — without it fail2ban may fail to parse dates and
  silently match nothing.
