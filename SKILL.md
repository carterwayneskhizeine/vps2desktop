---
name: vps2desktop
description: Protocol for deploying and maintaining Ubuntu x86_64 VPS machines as root-managed, RDP-capable desktops, driven by a per-machine manifest in machines/. Follow this file whenever a user asks to deploy, inspect, or update a machine.
---

# vps2desktop — Agent Protocol

Three modes:

| Mode | Invocation | What it does |
|---|---|---|
| **deploy** | "deploy `machines/<alias>`" | preflight → rootify → install + verify each component → record |
| **status** | "status `machines/<alias>`" | connect, report installed versions vs manifest, list drift |
| **update-check** | "check updates for `machines/<alias>`" | diff `catalog/CHANGELOG.md` past `last_sync`, offer new/changed components |

## Ground rules — never break these

1. **Credentials** resolve in this order: `manifest.yaml → credentials:` block → ask the user in chat. Never echo a password into output or logs; never write it anywhere outside `machines/`.
2. **Target must be Ubuntu ≥ 24.04 on x86_64** (`/etc/os-release`, `uname -m`). Anything else: stop and tell the user. No best-effort installs on other distros/arches.
3. **Always latest**: every component installs the latest version from official sources *at install time*. No pinned versions anywhere. Channel specifics live in each component doc.
4. **Idempotent**: run each component's *Guard* first; if already present and verify passes, record as `skipped-existing` and move on.
5. **No verify, no record**: an install whose *Verify* does not pass is a failure — stop, report, record nothing.
6. **Never automate login states or API keys** (OAuth flows, aichat `config.yaml`, tavily key, gh auth). Collect them into a *manual follow-ups* list instead.
7. **Never commit anything under `machines/`** (ADR-0002). Never print credentials.
8. **Install only what the manifest selects**, plus dependencies auto-added per `catalog/INDEX.yaml` — and say out loud when you auto-add one.

## Connection discipline

**One multiplexed connection for the whole session.** fail2ban on the target may ban a source IP that opens many *new* SSH connections in a short window — even with successful auth.

```bash
SSHCTL="$HOME/.ssh/vps2desktop-<alias>"
ssh -o ControlMaster=yes -o ControlPath="$SSHCTL" -o ControlPersist=30m \
    -o StrictHostKeyChecking=accept-new -p <port> <user>@<host> true   # open once
ssh -o ControlPath="$SSHCTL" -p <port> <user>@<host> '<command>'       # reuse for everything
```

- **Non-interactive shells do not source `.bashrc`**: nvm, conda, and `~/.local/bin` are NOT on PATH. Prefix remote commands with `export PATH=/root/.local/bin:$PATH` plus `for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done`, or use absolute paths — in *Install* and *Verify* alike. (Globs do not expand inside a `PATH=` assignment: a one-line `export PATH=…node/*/bin:$PATH` silently keeps the literal `*`.)
- **Password auth without leaking it**: write an askpass helper (`SSH_ASKPASS` + `SSH_ASKPASS_REQUIRE=force`, mode 600, deleted afterwards), or `sshpass -e` with the password in an environment variable. Never `sshpass -p`, never an inline argument.
- For sudo on the vendor user: `printf '%s\n' "$PASS" | sudo -S -p '' <cmd>`.

## Deploy mode

### 1. Preflight

1. Read `machines/<alias>/manifest.yaml`. Resolve credentials (rules above).
2. Open the control connection.
3. Checks: `source /etc/os-release && echo "$ID $VERSION_ID"` and `uname -m` → must be `ubuntu` ≥ 24.04, `x86_64`. `df -h /` for headroom. Note what already exists under `/root` (baseline for the profile).
4. `apt-get update` once (DEBIAN_FRONTEND=noninteractive everywhere).

### 2. Rootify (ADR-0003)

If the login user is already root → done.

1. Set root's password to the same one the user gave (three stdin lines: sudo's ask, then the two `passwd` asks):
   ```bash
   printf '%s\n%s\n%s\n' "$PASS" "$PASS" "$PASS" | sudo -S -p '' passwd root
   ```
2. `printf 'PermitRootLogin yes\n' | sudo -S -p '' tee /etc/ssh/sshd_config.d/20-vps2desktop-root.conf`, then `sshd -t`, then `systemctl reload ssh` (reload never drops existing sessions).
3. **Verify root login works** over a new control connection as root before giving up the vendor session.
4. Continue as root. Leave the vendor user untouched — it is the fallback if anything goes wrong.

### 3. Install loop

Order components by `catalog/INDEX.yaml` (`deps` first; announce auto-added dependencies).

For each component id:

1. Read `catalog/<id>.md` in full.
2. Run its **Guard** — present + verify passes → record `skipped-existing`, next.
3. Run its **Install** (absolute paths / explicit PATH; `curl --fail --location --retry 3` for downloads).
4. Run its **Verify** — must pass.
5. Append to the manifest's `installed:` list: `id`, `version` (captured by verify), `date`. Update the profile table as you go.

On failure: **stop the whole loop.** Report the component, the failing command, and the error; point at the component's *Known Pitfalls* section; ask the user how to proceed. Do not skip-and-continue silently.

### 4. Post-deploy

1. Write `machines/<alias>/profile.md` from `templates/machine/profile.md`: connection facts, installed table, manual follow-ups.
2. Print the follow-up list to the user — typically: RDP address/port and that the RDP password equals the root password; fcitx5 needs one logout/login; OAuth logins for claude/codex/opencode; aichat needs `~/.config/aichat/config.yaml`; tavily needs its key.

## Status mode

Connect, run each installed component's *Verify* to capture current versions, diff against the manifest's `installed:` list, report drift. Version drift upward is expected (always-latest); *missing* components are problems.

## Update-check mode

1. Ask the user, then `git pull` this repository.
2. Read `catalog/CHANGELOG.md`; every entry has an id (`c1`, `c2`, …). Take entries newer than the manifest's `last_sync` (empty `last_sync` = everything).
3. Compare with the manifest's `installed:` list → new components the user doesn't have, and changed components they do have.
4. Offer each and let the user choose; run the normal install loop for what they accept.
5. Set `last_sync` to the newest CHANGELOG entry id applied or seen.
