# Machine profile — <alias> — LOCAL ONLY (machines/ is gitignored)

| Field | Value |
|---|---|
| Alias | <alias> |
| Provider / region | |
| Host / port | <host> : <port> |
| Login | root (password — see manifest) |
| OS / arch | Ubuntu <version> / x86_64 |
| Deployed | <date> |
| Last status check | <date> |

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
