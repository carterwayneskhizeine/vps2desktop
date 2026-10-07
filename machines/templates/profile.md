# Machine profile — <alias> — shared template

Copy to `machines/<alias>/profile.md` before filling in machine details.
The copy is Local State and gitignored; `templates` is a reserved Machine Alias.

| Field | Value |
|---|---|
| Alias | <alias> |
| Hostname | <hostname> |
| Machine ID | <value from /etc/machine-id on the verified target> |
| Provider / region | |
| Host / port | <host> : <port> |
| Login | root (password — see manifest) |
| OS / arch | Ubuntu <version> / x86_64 |
| Deployed | <date> |
| Last status check | <date> |

Hostname and Machine ID identify the target; compare them with the current
environment before opening SSH. Record actual values only in the copied
Machine Profile, after verifying the target. A rebuild can change Machine ID;
confirm the new identity before updating it. Local versus remote execution is
determined each session and is not a fixed profile field.

## Installed

| Component | Version | Installed | Notes |
|---|---|---|---|
| base-tools | (apt set) | <date> | |

## Manual follow-ups

- [ ] RDP: connect with your client to <host>:3389, user `root`, same password as SSH
- [ ] fcitx5: log out/in once inside the RDP session before first use
- [ ] OAuth logins: claude / codex / opencode
- [ ] aichat: place `~/.config/aichat/config.yaml`
- [ ] tavily: set API key (`tvly` reads it from the environment)

## Notes

- ...
