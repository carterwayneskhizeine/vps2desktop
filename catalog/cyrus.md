# Cyrus

| | |
|---|---|
| id | `cyrus` |
| group | ai-cli |
| deps | `base-tools`, `node-nvm`, `claude-code` |
| channel | npm (`cyrus-ai@latest`) |

Cyrus connects issue trackers and chat platforms to coding agents. This Component
documents the community self-hosted path: Linear delegates a task, Cyrus creates
a Git worktree, Claude processes it, and activities/results return to Linear.
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
command -v cyrus >/dev/null 2>&1 &&
  test -f /root/.cyrus/.env &&
  test -f /root/.cyrus/config.json
```

Run Verify before recording `skipped-existing`. A running, configured installation
does not need to be reinstalled merely to document it.

## Install

```bash
export PATH="/root/.local/bin:$PATH"
for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done
npm install -g cyrus-ai@latest

umask 077
mkdir -p /root/.cyrus
touch /root/.cyrus/.env
if [ ! -f /root/.cyrus/config.json ]; then
  printf '{"repositories":[]}\n' > /root/.cyrus/config.json
fi
chmod 700 /root/.cyrus
chmod 600 /root/.cyrus/.env /root/.cyrus/config.json
```

The initialization commands are normally silent. They preserve existing config
and credentials. Seed `config.json` before `self-auth-linear`: that command reads
an existing config file, and running the worker first is not a reliable way to
create it on every release.

## Verify

```bash
export PATH="/root/.local/bin:$PATH"
for d in /root/.nvm/versions/node/*/bin; do PATH="$d:$PATH"; done
cyrus --version
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

Record the reported CLI version after these checks pass. This verifies the
installation; record integration readiness separately after the manual checks
below. Never dump `.env`, `config.json`, tokens or private keys into agent output.

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

- **npm ≥ 12 blocks postinstall scripts, including transitive ones.** `cyrus-ai`
  pulls `cloudflared`, whose postinstall fetches the cloudflared binary; npm
  warns `install-scripts … cloudflared@… (postinstall: node scripts/postinstall.mjs
  && node lib/cloudflared.js bin install)` and skips it. Rerun with
  `--allow-scripts=cyrus-ai --allow-scripts=cloudflared`. A clean install shows
  no `npm warn install-scripts` lines. The binary lands inside the package
  directory, not on `PATH` — that is expected; the Component's tunnel is
  external.
- **`Invalid webhook signature`**: verify that the app's Webhook Signing Secret
  was copied into `LINEAR_WEBHOOK_SECRET`, with no placeholder or accidental
  whitespace. OAuth authorization can succeed while this independent secret is
  wrong. Restart the worker after changing it: the direct webhook client captures
  the signing secret at startup. Do not disable verification to make events pass.
- **Only `Issue/update` after changing labels**: webhook delivery is working,
  but a label change only affects routing. Delegate the issue to the Cyrus app
  (or use a supported Agent mention flow) to create an Agent Session. If a prior
  delegation was rejected, re-delegation can provide a fresh test event.
- **`check-tokens` never returns**: observed in v0.2.73. The command leaves its
  `.env` watcher open, and older logic checks only workspaces referenced by
  repositories, so checking before adding a repo may print no result. Bound the
  diagnostic with `timeout --signal=INT --kill-after=3s 20s cyrus check-tokens`;
  timeout exit code 124 alone does not prove invalid OAuth credentials. A local
  package repair is not an upstream release and may be overwritten by reinstall.
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
