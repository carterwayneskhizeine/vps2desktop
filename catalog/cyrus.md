# Cyrus

| | |
|---|---|
| id | `cyrus` |
| group | ai-cli |
| deps | `base-tools`, `node-nvm`, `claude-code` |
| channel | source (`github.com/cyrusagents/cyrus`, `pnpm build` + `npm link`) |

Cyrus connects issue trackers and chat platforms to coding agents. This Component
documents the community self-hosted path: Linear delegates a task, Cyrus creates
a Git worktree, a configured runner processes it — Claude Code by default, with
Codex and other runners selectable per issue — and activities/results return to
Linear.
GitHub CLI authentication lets the agent push branches and create PRs. GitHub
App/webhook setup is a separate, optional integration for inbound GitHub events.

The dependencies cover this Claude-based path. Cyrus also supports other runners;
configure their credentials separately when selecting one. Installation and local
initialization are automated; account login, app creation, secret entry and OAuth
authorization remain manual follow-ups.

Official references: [Cyrus](https://github.com/cyrusagents/cyrus),
[self-hosting](https://github.com/cyrusagents/cyrus/blob/main/docs/SELF_HOSTING.md),
[configuration](https://github.com/cyrusagents/cyrus/blob/main/docs/CONFIG_FILE.md),
[Linear app manifest setup](https://github.com/cyrusagents/cyrus/blob/main/skills/cyrus-setup-linear/SKILL.md).

## Guard (idempotency)

```bash
export PATH="/root/.local/bin:$PATH"
for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done
CYRUS_SRC="${CYRUS_SRC:-/root/Code/cyrus}"
test -d "$CYRUS_SRC/.git" &&
  test -f "$CYRUS_SRC/apps/cli/dist/src/app.js" &&
  test "$(readlink -f "$(command -v cyrus)")" = "$(readlink -f "$CYRUS_SRC/apps/cli/dist/src/app.js")" &&
  test -f /root/.cyrus/.env &&
  test -f /root/.cyrus/config.json
```

Run Verify before recording `skipped-existing`. A running, configured installation
does not need to be reinstalled merely to document it.

## Install

```bash
export PATH="/root/.local/bin:$PATH"
for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done
CYRUS_SRC="${CYRUS_SRC:-/root/Code/cyrus}"

if [ -d "$CYRUS_SRC/.git" ]; then
  git -C "$CYRUS_SRC" pull --ff-only
else
  git clone https://github.com/cyrusagents/cyrus.git "$CYRUS_SRC"
fi

(cd "$CYRUS_SRC" && pnpm install && pnpm build)

# Replace any npm-global cyrus-ai with a symlink to this checkout.
npm uninstall -g cyrus-ai >/dev/null 2>&1 || true
(cd "$CYRUS_SRC/apps/cli" && npm link)

umask 077
mkdir -p /root/.cyrus
touch /root/.cyrus/.env
if [ ! -f /root/.cyrus/config.json ]; then
  printf '{"repositories":[]}\n' > /root/.cyrus/config.json
fi
chmod 700 /root/.cyrus
chmod 600 /root/.cyrus/.env /root/.cyrus/config.json
```

The source install builds every workspace package with pnpm and links `apps/cli`
into the npm global prefix. The `node-nvm` dependency supplies Node, npm and
pnpm; `base-tools` supplies git. `npm link` is deliberate — `pnpm link -g` cannot
link this monorepo (see Known Pitfalls). The runtime state under `/root/.cyrus`
is not touched by the switch, so an existing Linear authorization and repository
setup survive it. Stop a running worker before running the Install block: it
replaces the global `cyrus` on disk while the old process still has the previous
code in memory.

The initialization commands are normally silent. They preserve existing config
and credentials. Seed `config.json` before `self-auth-linear`: that command reads
an existing config file, and running the worker first is not a reliable way to
create it on every release.

## Verify

```bash
export PATH="/root/.local/bin:$PATH"
for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done
CYRUS_SRC="${CYRUS_SRC:-/root/Code/cyrus}"
cyrus --version
test "$(readlink -f "$(command -v cyrus)")" = "$(readlink -f "$CYRUS_SRC/apps/cli/dist/src/app.js")"
test "$(stat -c %a /root/.cyrus)" = 700
test "$(stat -c %a /root/.cyrus/.env)" = 600
test "$(stat -c %a /root/.cyrus/config.json)" = 600
node --input-type=module <<'NODE'
import { readFileSync } from 'node:fs';
const config = JSON.parse(readFileSync('/root/.cyrus/config.json', 'utf8'));
if (!Array.isArray(config.repositories)) {
  throw new Error('config.json must contain a repositories array');
}
console.log('Cyrus config JSON is valid');
NODE
```

Record the reported CLI version after these checks pass. The path assertion
distinguishes the source build from a leftover npm global package. This verifies
the installation; record integration readiness separately after the manual checks
below. Never dump `.env`, `config.json`, tokens or private keys into agent output.

## Updating a source install

```bash
export PATH="/root/.local/bin:$PATH"
for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done
CYRUS_SRC="${CYRUS_SRC:-/root/Code/cyrus}"
git -C "$CYRUS_SRC" pull --ff-only
(cd "$CYRUS_SRC" && pnpm install && pnpm build)
```

Then restart the worker by hand (see §5 for the foreground process; stop it
first). A running process keeps the code it loaded at startup, so `git pull`
alone never changes behavior. `npm link` does not need repeating unless the
package's `bin` entry changes.

## Runners and permission posture

Cyrus runs one runner per Agent Session. `CYRUS_DEFAULT_RUNNER` in `.env` picks
the default (`claude`); per-issue overrides win in this order: description tags
(`[agent=codex] [model=gpt-6.1-sol]`) > provider/model labels > agent labels >
model labels > repository `model` > per-runner `codexDefaultModel`-style
defaults. A description tag silently beats the default runner — delete the tag
from the issue when it should fall back to `claude`.

### Codex runner

- `codexSandboxMode` in `config.json` (`read-only`, `workspace-write`,
  `danger-full-access`) is passed per session; hot reload applies it without a
  restart. Default is `workspace-write`.
- The Codex CLI reads `~/.codex/config.toml`, but a Cyrus-spawned session
  receives its sandbox per thread, so `sandbox_mode` there does not apply to
  Cyrus sessions. Approval is always `never` for spawned sessions.
- Verify a standalone config with `codex doctor`; it prints the effective
  sandbox and network posture.

### Claude runner

- Cyrus loads user settings (`settingSources` includes `user`) and inherits the
  full parent environment, so `~/.claude/settings.json` governs Cyrus sessions
  exactly as interactive ones — auth, model and `permissions` included.
- Full-permission path: `permissions.defaultMode: "bypassPermissions"` in
  settings.json plus `IS_SANDBOX=1` in the worker environment. Under root the
  settings key alone is rejected at startup (see `claude-code` Known
  Pitfalls); the worker must be started with `IS_SANDBOX=1` already exported.
- The home-directory Read deny list is code-level: everything under `~/`
  outside the worktree, attachments and repository paths is denied — and
  current Claude Code extends `Read` deny rules to file-reading Bash commands
  (`ls`, `cat`), resolves symlinks to their real path before applying rules,
  and lets explicit deny rules win over bypass mode. Only allow-listing a
  directory lifts it.
- `claudeAllowedDirectories` in `config.json` whitelists extra directories for
  every Claude session: entries are `~`-expanded, granted as `Read(dir/**)`,
  and carved out of the home-directory deny list. Hot reload; removing the key
  restores the default.

## Manual follow-ups

### 1. Publish the webhook endpoint

Choose an existing reverse proxy or persistent tunnel and a public HTTPS origin,
such as `https://cyrus.example.com`, with no trailing slash. Route it to port
`3456` on the machine running Cyrus. The callback and webhook endpoints must be
reachable without an interactive sign-in to a proxy access page.

For a **Docker-hosted Cloudflare Tunnel on Linux**, a Published application can
use `http://host.docker.internal:3456`. The tunnel container needs this Compose
setting (merge it into the existing service):

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

Cyrus must listen on an address reachable from that container;
`CYRUS_HOST_EXTERNAL=true` selects `0.0.0.0`. A native tunnel on the host can
reach a localhost origin. Do not assume Docker Desktop's hostname mapping exists
on Docker Engine without the explicit host-gateway mapping.

A 502 while nothing listens on `3456` does not establish a broken tunnel. Check
the listener first; test the public URL while the callback or worker is running.

### 2. Fill the runtime environment manually

The user edits `/root/.cyrus/.env`. Seed only non-secret values and empty fields;
the agent must not collect or paste credentials. Keep real deployment facts in
Local State and real runtime credentials in the user-managed application files.

```dotenv
CYRUS_BASE_URL=https://cyrus.example.com
CYRUS_SERVER_PORT=3456
CYRUS_HOST_EXTERNAL=true
LINEAR_DIRECT_WEBHOOKS=true
CYRUS_DEFAULT_RUNNER=claude

LINEAR_CLIENT_ID=
LINEAR_CLIENT_SECRET=
LINEAR_WEBHOOK_SECRET=

# Choose the appropriate Claude authentication method:
# ANTHROPIC_API_KEY=
# CLAUDE_CODE_OAUTH_TOKEN=
```

The user can run `claude auth status` to inspect existing authentication, or
`claude setup-token` to generate a subscription token for manual entry. An API
key is another option. Do not capture a generated token into the conversation.
Verify authentication in the same account/environment that will run Cyrus;
systemd does not inherit temporary exports from an interactive terminal.

### 3. Create a Linear OAuth app

Use a Linear workspace administrator account. Browser login is used to create
and authorize the app; Cyrus runs with stored OAuth access/refresh tokens. A
personal Linear API key or browser cookies do not replace this Agent app setup.

Have the user substitute the HTTPS origin below, run the command, open the
printed manifest-backed URL, review the pre-filled settings, and create the app:

```bash
CYRUS_SETUP_BASE_URL="https://cyrus.example.com" node <<'NODE'
const base = process.env.CYRUS_SETUP_BASE_URL.replace(/\/+$/, '');
const manifest = {
  $schema: 'https://linear.app/.well-known/oauth-app-manifest.schema.json',
  schemaVersion: '1.0.0',
  distribution: 'private',
  display: { description: 'AI coding agent for automated development' },
  developer: { name: 'Self-hosted' },
  oauth: {
    client_name: 'Cyrus',
    client_uri: 'https://github.com/cyrusagents/cyrus',
    redirect_uris: [base + '/callback'],
    grant_types: ['authorization_code']
  },
  webhook: {
    enabled: true,
    url: base + '/linear-webhook',
    resourceTypes: ['AgentSessionEvent', 'AppUserNotification', 'PermissionChange', 'Issue']
  }
};
console.log('https://linear.app/settings/api/applications/new?manifest=' +
  encodeURIComponent(JSON.stringify(manifest)));
NODE
```

The user copies **Client ID**, **Client Secret**, and **Webhook Signing Secret**
to the corresponding `.env` fields. The signing secret belongs specifically in
`LINEAR_WEBHOOK_SECRET`; it is different from the OAuth Client Secret.

Run the callback flow with the main worker stopped, since both use port `3456`:

```bash
cyrus self-auth-linear
```

The user opens the displayed authorization URL in their signed-in browser and
authorizes the app. The browser can be on another computer: the public HTTPS
callback returns to the VPS. Success saves workspace access/refresh tokens in
`/root/.cyrus/config.json`. Do not print that file to verify success.

### 4. Connect a repository

GitHub authentication is needed for private GitHub repos, pushes and PR creation.
Check before asking the user to log in again:

```bash
gh auth status
git config --global user.name
git config --global user.email
```

If missing, the user completes `gh auth login --hostname github.com
--git-protocol https --web` and sets their preferred Git commit name/email.
For HTTPS Git operations, configure the credential helper:

```bash
gh auth setup-git --hostname github.com
cyrus self-add-repo https://github.com/example-org/example-repo.git
```

Replace the example repo URL. This clones under `/root/.cyrus/repos/`, associates
the repo with an authorized Linear workspace, detects its base branch, and sets
the repo name as its default routing label. A GitHub App is not needed merely
for creating PRs through the authenticated `gh` account.

### 5. Start and verify the integration

```bash
cyrus
```

Keep that foreground process running. In another terminal:

```bash
curl --fail --silent --show-error --max-time 10 http://127.0.0.1:3456/status
curl --fail --silent --show-error --max-time 10 https://cyrus.example.com/status
```

Substitute the actual public origin. Both should return JSON with `status` of
`idle` or `busy`. `idle` means the service is running and waiting; it does not
prove an issue has been processed or that an incoming signature is correct.

For the functional check, create a small read-only Linear issue, such as asking
for `pwd` and a machine summary. Add the repo's routing label and **delegate it
to Cyrus** through Linear's assignment menu. Labels and ordinary `Issue/update`
webhooks alone do not start an Agent Session. Expect `AgentSessionEvent`, a task
worktree, an AI session, a posted result, and a successful session completion.

When inspecting a large captured log, select the lifecycle entries rather than
copying the complete log into a report:

```bash
rg -n 'session_started|session_completed|Session completed|Result message emitted' /path/to/cyrus.log | tail -n 20
```

### 6. Optional persistence with systemd

After a successful foreground test, stop that worker before starting a service
on the same port. Resolve paths in the terminal where Node, jq and Cyrus work;
do not copy a hardcoded nvm version from another machine:

```bash
CYRUS_EXECUTABLE="$(command -v cyrus)"
CYRUS_NODE_BIN_DIR="$(dirname "$(command -v node)")"
CYRUS_JQ_BIN_DIR="$(dirname "$(command -v jq)")"
test -x "$CYRUS_EXECUTABLE"

cat > /etc/systemd/system/cyrus.service <<EOF
[Unit]
Description=Cyrus AI Agent
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=root
WorkingDirectory=/root
Environment="PATH=$CYRUS_NODE_BIN_DIR:$CYRUS_JQ_BIN_DIR:/root/.local/bin:/usr/local/bin:/usr/bin:/bin"
EnvironmentFile=/root/.cyrus/.env
ExecStart=$CYRUS_EXECUTABLE
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now cyrus.service
systemctl is-active cyrus.service
curl --fail --silent --show-error --max-time 10 http://127.0.0.1:3456/status
```

Regenerate the resolved paths if changing the nvm installation used for Cyrus.
Record persistence only after verifying that service; a working foreground
process is not proof of reboot persistence.

## Known Pitfalls

- **pnpm skips dependency build scripts by default.** `pnpm install` reports
  `The following dependencies have build scripts that were ignored: cloudflared`.
  pnpm 10 runs only the packages allowlisted under `onlyBuiltDependencies` in
  `pnpm-workspace.yaml`, and the bundled cloudflared binary is irrelevant here
  because the Component's tunnel is external. Leave it ignored; do not run
  `pnpm approve-builds` unless that binary is actually needed.
- **`pnpm link -g` cannot link this monorepo.** Two independent failures: a stock
  box has no pnpm global bin directory (`ERR_PNPM_NO_GLOBAL_BIN_DIR` — `pnpm
  setup` or `PNPM_HOME` would be required), and global-linking `apps/cli` tries
  to resolve its `workspace:*` dependencies outside the workspace
  (`ERR_PNPM_WORKSPACE_PKG_NOT_FOUND`). `npm link` from `apps/cli` creates only
  the global package directory and `cyrus` bin symlinks; the package resolves
  its dependencies from the checkout's own `node_modules`.
- **The `cyrus` command is coupled to the checkout path.** `npm link` points the
  global bin at `apps/cli/dist/src/app.js`; moving or deleting the checkout
  breaks the command, and every `git pull` needs `pnpm install && pnpm build`
  before the new code runs. Verify the resolved path, not just `--version`: both
  an old npm package and a source build report a version number.
- **`Invalid webhook signature`**: verify that the app's Webhook Signing Secret
  was copied into `LINEAR_WEBHOOK_SECRET`, with no placeholder or accidental
  whitespace. OAuth authorization can succeed while this independent secret is
  wrong. Restart the worker after changing it: the direct webhook client captures
  the signing secret at startup. Do not disable verification to make events pass.
  Also check for a glued file: appending to `.env` with `echo ... >>` when the
  file lacks a trailing newline concatenates the new key onto the last line and
  corrupts the secret while everything still parses — ensure the file ends with
  a newline before appending. `/proc/<pid>/environ` does not show
  dotenv-loaded values, so absence there proves nothing about the worker's
  effective configuration.
- **Only `Issue/update` after changing labels**: webhook delivery is working,
  but a label change only affects routing. Delegate the issue to the Cyrus app
  (or use a supported Agent mention flow) to create an Agent Session. If a prior
  delegation was rejected, re-delegation can provide a fresh test event.
- **`check-tokens` never returns**: observed in v0.2.73 and still present in the
  v0.2.74 source. The command leaves its `.env` watcher open, and its logic
  checks only workspaces referenced by repositories, so checking before adding a
  repo may print no result. Bound the diagnostic with
  `timeout --signal=INT --kill-after=3s 20s cyrus check-tokens`; timeout exit
  code 124 alone does not prove invalid OAuth credentials. A local package
  repair is not an upstream release; it lives in the checkout's source and is
  lost on a fresh clone or overwritten by an update that does not include it.
- **Check OAuth independently when that diagnostic hangs**: the following
  read-only check has a request deadline and never prints credentials. It checks
  the saved access token; it does not refresh an expired token.

  ```bash
  node --input-type=module <<'NODE'
  import { readFileSync } from 'node:fs';
  const config = JSON.parse(readFileSync('/root/.cyrus/config.json', 'utf8'));
  const workspaces = Object.entries(config.linearWorkspaces ?? {});
  if (!workspaces.length) throw new Error('No authorized Linear workspaces');
  for (const [id, workspace] of workspaces) {
    const response = await fetch('https://api.linear.app/graphql', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: 'Bearer ' + workspace.linearToken
      },
      body: JSON.stringify({ query: '{ viewer { id } }' }),
      signal: AbortSignal.timeout(10000)
    });
    const data = await response.json();
    if (!response.ok || data.errors || !data.data?.viewer?.id) {
      throw new Error('Linear token check failed; HTTP ' + response.status);
    }
    console.log((workspace.linearWorkspaceName || id) + ': token valid');
  }
  NODE
  ```

- **`Finished` with the issue still `In Progress`**: Agent Session completion and
  issue workflow status are separate. The current default moves an accepted task
  to a `started` state and publishes the result when the session completes; it
  does not automatically mark the issue Done or In Review. A terminal issue state
  can stop sessions and remove worktrees. Define any additional automation
  deliberately; do not invent an undocumented `config.json` flag.
- **Where workflow names and `type` come from**: Linear's team settings, not the
  project or Cyrus config. Open **Settings -> Teams -> <team> -> Issue statuses**.
  A state created in **Started** has `type: started`; one in **Completed** has
  `type: completed`. Names such as In Progress, Human Review or In Review can
  differ between teams. Cyrus selects the first state in the `started` category
  when beginning work. See [Linear's workflow documentation](https://linear.app/docs/configuring-workflows).
- **Cannot attach to a signed-in Chrome**: `agent-browser --auto-connect`
  requires a reachable running Chrome with debugging enabled. A browser signed
  in on the user's laptop is not automatically available to a CLI on the VPS.
  Have the user open the displayed URLs themselves; do not extract cookies,
  tokens or browser profiles to bypass that boundary.
