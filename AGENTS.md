# AGENTS.md

Entry point for any AI agent (Claude Code, Codex, OpenCode, pi, ...) working in this repository.

1. **Read [`SKILL.md`](SKILL.md) first** — it is the protocol for deploying and maintaining VPS machines. Do not improvise installation steps; component docs in `catalog/` are the single source of truth for commands.
2. **Never commit anything under `machines/`** — it holds user credentials and machine state, and is gitignored by design (ADR-0002). Never print credentials into logs or command output.
3. **One SSH connection per session**: reuse a ControlMaster multiplexed connection for every remote command (see SKILL.md §Connection discipline).
4. Vocabulary is defined in [`CONTEXT.md`](CONTEXT.md) — use those terms exactly.
5. Component changes: update `catalog/INDEX.yaml` and append to `catalog/CHANGELOG.md` in the same commit.
