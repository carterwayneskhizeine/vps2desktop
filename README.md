# vps2desktop

[English](README.md) | [简体中文](README-zh.md)

Turn a fresh, SSH-only **Ubuntu x86_64 VPS** into a fully usable **remote desktop (RDP) machine** — with the AI agent of your choice doing the work.

You pick components on a checklist (Xfce desktop, Chinese input, browser, editors, AI CLIs...), hand the manifest plus connection info to any AI agent (Claude Code, Codex, OpenCode, pi...), and it turns the box into a root-managed, RDP-ready workstation with **always-latest versions** pulled from official sources at install time.

## How it works

```
1. git clone this repo
2. Open checklist.html in your browser
   → pick a preset or check components individually (dependencies auto-select)
   → optionally fill in connection info (host / port / user / password)
   → export manifest.yaml, move it to machines/<alias>/manifest.yaml
3. Tell your AI agent: "deploy machines/<alias>"
   → it follows SKILL.md: preflight → rootify → install each component → verify → record
```

For day-to-day updates: `git pull`, then ask your agent to *check updates* — it reads `catalog/CHANGELOG.md`, compares with your machine's manifest, and offers to install anything new.

`machines/` exists after cloning because it contains shared templates. To start from a template, copy `machines/templates/` to `machines/<alias>/` and edit the copied manifest, or save your checklist export there. Your machine files are gitignored. `templates` is reserved and cannot be used as a Machine Alias; keep the shared templates free of real credentials.

## What gets installed

Everything in `catalog/` is optional and selectable. Highlights: Xfce + xrdp with working audio (PipeWire), fcitx5 Chinese input, Chrome / VS Code (root-safe wrappers), Snipaste (screenshot, autostarts), PeaZip (with Thunar right-click actions), Warp terminal, nvm+Node LTS / uv / Anaconda, and the AI CLI set: Claude Code, Codex, OpenCode, pi, cc-switch, aichat, agent-browser, tavily.

## Requirements

- A VPS running **Ubuntu 24.04+ on x86_64** (anything else: the agent will refuse and tell you)
- Any AI coding agent that can read this repo and run `ssh` (Claude Code, Codex, OpenCode, pi, ...)
- A POSIX shell for the agent to run from (Linux / macOS / WSL / Git Bash)

## Security notes (read this)

- Deploying standardizes the machine on **root password SSH login** (see `docs/adr/0003-root-password-login.md`). The `fail2ban` component exists to mitigate this.
- Your credentials live only in `machines/<alias>/manifest.yaml`, which is **gitignored** and never leaves your machine. Nothing you type into `checklist.html` is sent anywhere — it is a single offline HTML file.
- The vendor-provided original user (e.g. `ubuntu`) is left untouched as a fallback.

## Repository layout

| Path | Purpose |
|---|---|
| `SKILL.md` | The protocol your AI agent follows (deploy / status / update-check) |
| `checklist.html` | Offline single-file component picker; exports your manifest |
| `catalog/` | One doc per component: install commands, guards, verification, known pitfalls |
| `catalog/INDEX.yaml` | Component registry: groups, dependencies, presets |
| `machines/<alias>/` | Your Local State (gitignored) |
| `machines/templates/` | Shared templates (tracked); creates `machines/` when cloned |
| `docs/adr/` | Architecture Decision Records |
| `CONTEXT.md` | Glossary of terms used across the repo |

## Contributing

Found a pitfall that would hit everyone? Please open an issue or PR to improve the component doc in `catalog/` — see `CONTRIBUTING.md`. Machine-specific quirks belong in your local `machines/<alias>/notes.md`, not in the repo.

## License

[MIT](LICENSE)
