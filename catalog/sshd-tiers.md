# sshd-tiers (slow / daily sshd jails + recidive)

| | |
|---|---|
| id | `sshd-tiers` |
| group | base |
| deps | [fail2ban] |
| channel | apt |

The stock `sshd` jail only catches bursts. fail2ban 1.1.0 normalizes the
retry counter against the findtime window (`FailTicket.adjustTime`), so a
scanner that trickles one attempt every few minutes settles just below
maxretry and is never banned — even at thousands of attempts per day. This
component layers two quota jails and a repeat-offender jail on top of the
`fail2ban` component:

- `sshd-slow` — 8 attempts / 1 h → 6 h ban. Catches the 5–10 minute drip.
- `sshd-daily` — 15 attempts / 1 d → 1 d ban. Catches scanners that stay
  under the per-window threshold in *every* window (classically
  `maxretry - 1` attempts per wave, waves spaced 40–100 minutes apart).
- `recidive` — 3 bans in 24 h → all-port ban for 1 week.
- `[DEFAULT] bantime.increment` — factor 2, capped at 4 weeks, so a
  returning scanner escalates instead of resetting to the base bantime
  after every unban.

## Guard (idempotency)

```bash
for j in sshd-slow sshd-daily recidive; do
  fail2ban-client status "$j" >/dev/null 2>&1 || exit 1
done
[ "$(fail2ban-client get recidive bantime)" = 604800 ] && echo present
```

## Install

Edit `/etc/fail2ban/jail.local`. Keep a single `[DEFAULT]` section — merge
these keys into the existing one instead of adding a second section:

```ini
[DEFAULT]
bantime.increment = true
bantime.factor    = 2
bantime.maxtime   = 4w
# ... keep the existing keys (ignoreip, banaction) ...
```

Append the three jails:

```ini
[sshd-slow]
enabled      = true
filter       = sshd
backend      = systemd
journalmatch = _SYSTEMD_UNIT=ssh.service + _COMM=sshd
port         = ssh
maxretry     = 8
findtime     = 1h
bantime      = 6h

[sshd-daily]
enabled      = true
filter       = sshd
backend      = systemd
journalmatch = _SYSTEMD_UNIT=ssh.service + _COMM=sshd
port         = ssh
maxretry     = 15
findtime     = 1d
bantime      = 1d

[recidive]
enabled  = true
maxretry = 3
findtime = 1d
bantime  = 1w
# recidive reads fail2ban's own ban log, not the journal — see Pitfalls.
backend  = auto
logpath  = /var/log/fail2ban.log
```

Give the failure history room for a 1-day window in
`/etc/fail2ban/fail2ban.local`:

```ini
[Definition]
dbpurgeage = 7d
```

Validate and load. A reload replays the log but does **not** flush existing
bans (they live in `/var/lib/fail2ban/fail2ban.sqlite3`):

```bash
fail2ban-client -t
fail2ban-client reload
```

## Verify

```bash
fail2ban-client status | grep -E 'sshd-slow|sshd-daily|recidive'
fail2ban-client get sshd-slow  maxretry     # 8
fail2ban-client get sshd-daily findtime     # 86400
fail2ban-client get recidive   bantime      # 604800
nft list chain inet f2b-table f2b-chain | grep -c 'addr-set-'
```

`recidive` legitimately starts at `Total failed: 0` and stays there until a
*new* ban is written to the log. It is broken only when it remains at 0
while `/var/log/fail2ban.log` holds recent `Ban` lines (see Pitfalls).

## Known Pitfalls

- **Judge attempts per findtime window, never per day.** The retry counter is
  normalized to the window above, so "N attempts per day" says nothing about
  whether a ban is due. The only correct question is how many attempts the IP
  makes inside any `findtime` window.
- **Bursts below the threshold are a second evasion shape.** A scanner that
  sends exactly `maxretry - 1` attempts in one TCP session and returns
  40–100 minutes later is legal in every window: averages and daily totals say
  "long banned", the jail never fires. The daily quota is the fix; lowering
  `maxretry` to catch it also catches real users — RDP/VPN clients reconnect
  in storms and mistyped passwords come in runs.
- **`[DEFAULT] backend = systemd` in any `jail.d/*.conf` infects every jail.**
  Panel integrations ship one. `recidive` then reads the journal of
  `fail2ban.service`, which contains only start/stop lines — never a `Ban`
  line — and stays at `Total failed: 0` forever while looking healthy. Pin
  `backend = auto` + `logpath = /var/log/fail2ban.log` as above; diagnose with
  `journalctl -u fail2ban.service --no-pager | wc -l` (single digits) against
  `wc -l /var/log/fail2ban.log` (thousands).
- **Enabling recidive replays the whole log.** It immediately bans every IP
  with ≥3 bans inside `findtime`, including bans that already expired. Before
  the reload, list the candidates and confirm none is yours:
  `grep 'Ban ' /var/log/fail2ban.log | awk '{print $NF}' | sort | uniq -c | sort -rn | head`
- **`ignoreip` is replaced, not merged.** A jail that declares `ignoreip`
  drops the `[DEFAULT]` list entirely. Repeat every owner address in such a
  jail, or the owner becomes bannable there.
- **Jails without `port` inherit `[DEFAULT] port = 0:65535`.** The nftables
  action then rejects *every* TCP port for that IP
  (`tcp dport 0-65535 ... reject`), not just 22 — an accidental all-port ban.
  Declare `port = ssh` on the tier jails; all-port escalation is `recidive`'s
  job. Anyone behind the same NAT as the banned IP loses every service too, so
  all-port bans are the reason owner ranges must be whitelisted.
- **A manual ban follows the jail's bantime, not the escalation policy.**
  `fail2ban-client set sshd banip <ip>` expires with that jail (1 h). To
  actually lock an IP out, also send it to recidive:
  `fail2ban-client set recidive banip <ip>` (all ports, 1 week).
- **`dbpurgeage` must exceed the longest `findtime`.** The default 1 d sits
  exactly on the boundary of the daily jails; use 7 d or recidive loses
  history across restarts.
- **Counting:** `sshd`, `sshd-slow` and `sshd-daily` all read the same sshd
  journal, so one attempt is one `Found` line per jail; `fail2ban.filter` and
  `fail2ban.observer` also log every event twice. For reports count `Ban`
  lines from `fail2ban.actions`, dedupe per (IP, timestamp), and skip
  `Restore Ban` — that is a restart replay, not a new ban.
