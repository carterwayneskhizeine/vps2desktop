# OpenWiki

| | |
|---|---|
| id | `openwiki` |
| group | ai-cli |
| deps | `node-nvm` |
| channel | npm |

OpenWiki is a CLI that runs a DeepAgents documentation agent to generate and
maintain a wiki-style documentation set for a codebase. It installs as an npm
global package on the Node runtime supplied by `node-nvm`; the package needs
Node >= 22.22.0. The launch banner reports `provider: OpenAI` and the
configured model id, so provider setup belongs to usage — a clean banner on a
fresh install does not mean the agent can run.

Official references: [OpenWiki](https://github.com/langchain-ai/openwiki),
[npm: openwiki](https://www.npmjs.com/package/openwiki).

## Guard (idempotency)

```bash
export PATH="/root/.local/bin:$PATH"
for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done
test -x "$(command -v openwiki)"
GROOT="$(npm root -g)"
node -e "const D=require(process.argv[1] + '/openwiki/node_modules/better-sqlite3'); new D(':memory:').prepare('select 1 as x').get()" "$GROOT"
```

The last check is the one that matters: it loads the native `better-sqlite3`
binding directly, so a blocked install script fails here instead of at first
use.

## Install

```bash
export PATH="/root/.local/bin:$PATH"
for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done
npm install -g --allow-scripts=better-sqlite3 openwiki
```

`--allow-scripts` must be repeated on every install and upgrade — without it
the transitive `better-sqlite3` dependency is left without its native binding
(see Known Pitfalls).

## Verify

```bash
export PATH="/root/.local/bin:$PATH"
for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done
openwiki --help 2>&1 | grep -m1 'OpenWiki v'
GROOT="$(npm root -g)"
node -e "const D=require(process.argv[1] + '/openwiki/node_modules/better-sqlite3'); console.log('better-sqlite3 OK:', JSON.stringify(new D(':memory:').prepare('select 1 as x').get()))" "$GROOT"
npm ls -g openwiki
```

The banner line (`>_ OpenWiki vX.Y.Z agent docs for codebases`) is the version
of record — `openwiki --version` is not a supported flag. All checks are
headless-safe. Record the banner version after these checks pass.

## Known Pitfalls

- **npm's allowScripts policy blocks the `better-sqlite3` install script** —
  the install "succeeds" with only an `npm warn install-scripts` warning and
  the native binding is missing until rerun with
  `--allow-scripts=better-sqlite3` (verified 2026-10-23 on the local
  machine). A clean install shows no `npm warn install-scripts` lines.
- `openwiki --version` prints `Unknown option: --version`; read the version
  from the `--help` banner instead.
- The install prints an `ERESOLVE overriding peer dependency` warning
  (`deepagents@1.13.2` peers on `langsmith <0.10.0`, openwiki ships
  `^0.10.5`); npm resolves it and the CLI works — do not try to fix it.
- The banner reports `provider: OpenAI` and a model id; installation does not
  configure a provider key, so a clean banner does not mean the agent can
  actually run.

## Manual follow-ups

### 1. Configure the LLM provider

The banner reports `provider: OpenAI`; openwiki was cataloged before provider
setup was exercised, so the exact key hand-over (environment variable vs.
config file) is not yet documented here — record it on first real use.
