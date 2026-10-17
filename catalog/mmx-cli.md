# mmx (MiniMax CLI)

| | |
|---|---|
| id | `mmx-cli` |
| group | ai-cli |
| deps | `node-nvm` |
| channel | npm (latest) |

`mmx` is MiniMax's official command-line client — covers text chat (MiniMax Messages
API), image/video generation (image-01, Hailuo-2.3, MiniMax-H3, S2V-01), TTS/ASR
(speech-2.8-hd, asr-1.0), vision (VLM describe), web search, file storage and quota.
Authentication is OAuth or API key, both manual.

## Guard (idempotency)

```bash
ls /root/.nvm/versions/node/*/bin/mmx >/dev/null 2>&1 && echo present
```

## Install

```bash
for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done
npm install -g mmx-cli@latest
```

## Verify

```bash
for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done
mmx --version
```

## Known Pitfalls

- npm ≥ 12 blocks postinstall scripts by default; if the install warns, rerun with
  `--allow-scripts=mmx-cli`.
- `mmx` is published under the npm package name **`mmx-cli`**; the CLI itself is the
  `mmx` binary. Don't search `npm install mmx` — that name is taken by something else.
- PATH is inherited from the installing shell, not from the user shell's `~/.bashrc`.
  New login shells must re-evaluate `/root/.nvm/versions/node/*/bin` before `mmx`
  resolves — same pattern as every other Node-based CLI in this catalog.
- `mmx video generate` is async — it returns a `task_id`; poll with `mmx video task get`
  and fetch the result with `mmx video download`. Synchronous HTTP responses only mean
  "task accepted", not "video ready".
- `mmx search` is bounded at 10 results per call with no pagination — change the search
  term, don't expect a page=2.
- `mmx speech synthesize` is capped at 10 000 characters per call; split longer
  scripts yourself.

## Manual follow-ups

- `mmx auth login` — OAuth or API key. Until this is done, every command 401s.
- `--region cn` selects MiniMax's China region; `--region global` (default) hits the
  global endpoint. Override per-call with `--base-url` if you're pointing at a private
  gateway.