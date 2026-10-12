# tavily-cli (tvly)

| | |
|---|---|
| id | `tavily-cli` |
| group | ai-cli |
| deps | `uv` |
| channel | uv-tool (latest) |

Tavily web-search CLI, installed as a uv tool.

## Guard (idempotency)

```bash
test -x /root/.local/bin/tvly && echo present
```

## Install

```bash
export PATH=/root/.local/bin:$PATH
uv tool install tavily-cli
```

## Verify

```bash
/root/.local/bin/tvly --version
```

## Known Pitfalls

- The executable is **`tvly`**, not `tavily` — do not "fix" the name mismatch.
- uv tool bin dir is `/root/.local/bin` (already on the absolute-path discipline).
- `--country` requires a valid country name (e.g. `China`), not an ISO code — `CN` fails with "Invalid country". Consult the Tavily API reference for the supported list before flagging the parameter.
- `tvly init [--no-browser]` is the official skill installer for agent hosts (Claude Code / Codex detected automatically). With `--no-browser` it skips the browser OAuth step and authenticates from the `TAVILY_API_KEY` env var instead. Run it after every CLI upgrade — bundled skills are pinned to the CLI release.
- Skill refresh cycle: `tvly update` then `tvly init` reinstalls the bundled skills at the new release's pin. A bare `tvly update` alone will not refresh skill contents.
- `tvly login --api-key <key>` persists the API key to `~/.tavily/config.json` (mode 600) as an alternative to the browser OAuth flow. Prefer it when an API key is already available — browser OAuth routes through Tavily's MCP endpoint and has documented compatibility gaps for some `search` and `research` options.
- `tvly auth` reports the loaded credential but masks the API key (e.g. `tvly-dev...H2H6`) — safe to include in logs as an existence check; useful template for the project's "never echo credentials" rule.

## Manual follow-ups

- Needs `TAVILY_API_KEY` in the environment. If the user provides the key, the agent may append `TAVILY_API_KEY=...` to `/etc/environment` (with the user's explicit go-ahead); otherwise instruct the user to add it themselves. The key never enters the repo.
