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

## Manual follow-ups

- Needs `TAVILY_API_KEY` in the environment. If the user provides the key, the agent may append `TAVILY_API_KEY=...` to `/etc/environment` (with the user's explicit go-ahead); otherwise instruct the user to add it themselves. The key never enters the repo.
