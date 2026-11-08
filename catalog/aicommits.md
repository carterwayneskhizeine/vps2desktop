# aicommits

| | |
|---|---|
| id | `aicommits` |
| group | ai-cli |
| deps | `node-nvm` |
| channel | npm (latest) |

[Nutlope/aicommits](https://github.com/Nutlope/aicommits) as a global npm package under the nvm-managed Node — AI-written commit messages from staged git changes. Works against any OpenAI-compatible endpoint via `provider=custom` (verified with DeepSeek).

## Guard (idempotency)

```bash
ls /root/.nvm/versions/node/*/bin/aicommits >/dev/null 2>&1 && echo present
```

## Install

```bash
for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done
npm install -g aicommits@latest
```

## Configure (manual — holds the API key)

`aicommits setup` / `aicommits model` write `~/.aicommits`, a plain `key=value`
file that holds the API key — keep it mode 600. Provider settings used here
(custom endpoint, DeepSeek):

```ini
provider=custom
OPENAI_BASE_URL=https://api.deepseek.com
OPENAI_API_KEY=<key>
OPENAI_MODEL=<model>
type=conventional
```

The same keys can be written non-interactively with `aicommits config set
KEY=value`. The `type` line is the default style for the `-t/--type` flag.

## Verify

```bash
for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done
aicommits --version
```

A real end-to-end check needs a scratch repo with a staged change: `aicommits`
must print a generated message and commit it after confirmation. This is also
the only test that proves the provider credentials work.

## Known Pitfalls

- `~/.aicommits` starts world-readable (0644) — on a root-managed multi-user
  VPS that exposes the API key to every account; `chmod 600` it right after
  setup.
- Nothing happens without staged changes: aicommits reads `git diff --staged`
  only. `-a` stages tracked files first; untracked files still need a manual
  `git add`.
- `-t/--type` defaults to `plain` per invocation; the stored `type=` is only a
  default — pass `-t conventional` (or store it in config) for the
  conventional style.
- `aicommits pr` (beta) pushes a branch diff into a PR — do not run it in a
  scratch repo; it creates real pull requests.
- `aicommits hook` installs a `prepare-commit-msg` hook into the *current*
  repo, making every `git commit` AI-written. Per-repo and easy to forget —
  install it only where wanted, uninstall with `aicommits hook uninstall`.
- Upgrades: `aicommits update` shells out to the package manager; under nvm the
  plain `npm install -g aicommits@latest` is the reliable path.

## Manual follow-ups

- Set the provider/model via `aicommits setup` (or hand-write `~/.aicommits`,
  then `chmod 600`).
- Optional: `aicommits hook` per repo for fully automatic commit messages.
