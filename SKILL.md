---
name: vps2desktop
description: Protocol for deploying and maintaining Ubuntu x86_64 VPS machines as root-managed, RDP-capable desktops, driven by a per-machine manifest in machines/. Follow this file whenever a user asks to deploy, inspect, or update a machine.
---

# vps2desktop — Agent Protocol

Three modes:

| Mode | Invocation | What it does |
|---|---|---|
| **deploy** | "deploy `machines/<alias>`" | preflight → rootify → install + verify each component → record |
| **status** | "status `machines/<alias>`" | identify target, report installed versions vs manifest, list drift |
| **update-check** | "check updates for `machines/<alias>`" | diff `catalog/CHANGELOG.md` past `last_sync`, offer new/changed components |

## Ground rules — never break these

1. **Credentials** resolve in this order: `manifest.yaml → credentials:` block → ask the user in chat. Never echo a password into output or logs; never write it anywhere outside `machines/`.
2. **Target must be Ubuntu ≥ 24.04 on x86_64** (`/etc/os-release`, `uname -m`). Anything else: stop and tell the user. No best-effort installs on other distros/arches.
3. **Always latest**: every component installs the latest version from official sources *at install time*. No pinned versions anywhere. Channel specifics live in each component doc.
4. **Idempotent**: run each component's *Guard* first; if already present and verify passes, record as `skipped-existing` and move on.
5. **No verify, no record**: an install whose *Verify* does not pass is a failure — stop, report, record nothing.
6. **Never automate login states or API keys** (OAuth flows, aichat `config.yaml`, tavily key, gh auth). Collect them into a *manual follow-ups* list instead.
7. **Never commit Local State under `machines/<alias>/`** (ADR-0002). Only shared templates in `machines/templates/` are tracked; never write credentials into them. Never print credentials.
8. **Install only what the manifest selects**, plus dependencies auto-added per `catalog/INDEX.yaml` — and say out loud when you auto-add one.
9. **Identify the current machine before SSH**: follow the identity check below for every mode that accesses a VPS. When already on the target, execute locally; do not SSH back into it merely to run commands.

## Connection discipline

### Identify the target first

Before opening any SSH connection:

1. Read the target Manifest and existing Machine Profile. Compare the local `/etc/machine-id`, `hostname` and interface addresses (`ip -j address show` or `hostname -I`) with the profile's `Machine ID`, `Hostname` and the manifest's host. Compare identity values privately; never print credentials or dump the manifest/profile into logs.
2. Treat blank identity fields and template placeholders as unrecorded. A nonempty matching `Machine ID` identifies the current VPS. If no Machine ID has been recorded, a target IP assigned to a local interface, together with compatible profile facts, can establish a local match. A Machine Alias, hostname match, repository path or loopback address alone is not sufficient evidence. Account for containers or separate network namespaces; do not treat a container as the host just because a name or mounted machine ID matches.
3. If the current environment is the target VPS, announce local execution and run the normal Guard, Install and Verify commands locally with the required privileges. Do not request SSH credentials or open a connection just to execute them. Perform OS, architecture and user checks locally as well.
4. If evidence establishes a remote target, open the multiplexed connection below. Once connected, verify its identity against the Machine Profile before making changes. If identity evidence is ambiguous or contradicts the profile, stop and clarify the target before SSH or any changes; do not silently overwrite a stored Machine ID.
5. After confirming the target, record its `Hostname` (`hostname`) and `Machine ID` (`/etc/machine-id`) in the local Machine Profile. Refresh the hostname after a verified rename; after a rebuild, confirm the new identity before replacing the old Machine ID. Leave unavailable values blank. Never copy these actual values into shared templates. Decide local versus remote anew each session; do not store a fixed execution mode in the profile.

### Remote execution

**One multiplexed connection for the whole session when the target is remote.** fail2ban on the target may ban a source IP that opens many *new* SSH connections in a short window — even with successful auth.

```bash
SSHCTL="$HOME/.ssh/vps2desktop-<alias>"
ssh -o ControlMaster=yes -o ControlPath="$SSHCTL" -o ControlPersist=30m \
    -o StrictHostKeyChecking=accept-new -p <port> <user>@<host> true   # open once
ssh -o ControlPath="$SSHCTL" -p <port> <user>@<host> '<command>'       # reuse for everything
```

- **Non-interactive shells do not source `.bashrc`**: nvm, conda, and `~/.local/bin` are NOT on PATH. Prefix remote commands with `export PATH=/root/.local/bin:$PATH` plus `for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done`, or use absolute paths — in *Install* and *Verify* alike. (Globs do not expand inside a `PATH=` assignment: a one-line `export PATH=…node/*/bin:$PATH` silently keeps the literal `*`.)
- **Password auth without leaking it**: write an askpass helper (`SSH_ASKPASS` + `SSH_ASKPASS_REQUIRE=force`, mode 600, deleted afterwards), or `sshpass -e` with the password in an environment variable. Never `sshpass -p`, never an inline argument.
- For sudo on the vendor user: `printf '%s\n' "$PASS" | sudo -S -p '' <cmd>`.
- **Windows Git Bash: ControlMaster does not work** (`mux_client_request_session: read from master failed` — a client-side limitation, seen against two different targets). Fall back to one `ssh ... 'bash -s'` session per batch with the askpass helper; keep batches coarse (one component or one check group per connection) so the total number of connections stays low. A small per-machine runner script that re-reads the password from the manifest at runtime (see a machine's `.sshrun.sh`) keeps this disciplined.

## Deploy mode

### 1. Preflight

1. Read `machines/<alias>/manifest.yaml` and the existing Machine Profile. The Machine Alias `templates` is reserved for shared templates and must not be used for a deployment.
2. Follow the identity check in Connection discipline. Execute locally if already on the target; otherwise resolve credentials (rules above), open the control connection and confirm the remote identity.
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

1. Write `machines/<alias>/profile.md` from `machines/templates/profile.md`: connection facts, verified Hostname and Machine ID, installed table, manual follow-ups. Preserve existing identity fields unless a verified rename or confirmed rebuild requires an update.
2. Print the follow-up list to the user — typically: RDP address/port and that the RDP password equals the root password; fcitx5 needs one logout/login; OAuth logins for claude/codex/opencode; aichat needs `~/.config/aichat/config.yaml`; tavily needs its key.

## Status mode

Follow the identity check in Connection discipline first. Run each installed component's *Verify* locally when already on the target, or through the shared SSH connection when remote, to capture current versions. Record the verified Hostname and Machine ID in the Machine Profile, diff against the manifest's `installed:` list, and report drift. Version drift upward is expected (always-latest); *missing* components are problems.

## Update-check mode

1. Ask the user, then `git pull` this repository.
2. Read `catalog/CHANGELOG.md`; every entry has an id (`c1`, `c2`, …). Take entries newer than the manifest's `last_sync` (empty `last_sync` = everything).
3. Compare with the manifest's `installed:` list → new components the user doesn't have, and changed components they do have.
4. Offer each and let the user choose; run the normal install loop for what they accept.
5. Set `last_sync` to the newest CHANGELOG entry id applied or seen.
