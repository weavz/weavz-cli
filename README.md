# Weavz CLI

[![npm version](https://img.shields.io/npm/v/%40weavz-io%2Fcli?label=npm)](https://www.npmjs.com/package/@weavz-io/cli)
[![npm downloads](https://img.shields.io/npm/dm/%40weavz-io%2Fcli?label=downloads)](https://www.npmjs.com/package/@weavz-io/cli)
[![GitHub release](https://img.shields.io/github/v/release/weavz/weavz-cli?label=release)](https://github.com/weavz/weavz-cli/releases)
[![Release workflow](https://github.com/weavz/weavz-cli/actions/workflows/release.yml/badge.svg?branch=main)](https://github.com/weavz/weavz-cli/actions/workflows/release.yml)
[![License](https://img.shields.io/npm/l/%40weavz-io%2Fcli?label=license)](https://github.com/weavz/weavz-cli/blob/main/LICENSE)

Official command-line interface for [Weavz](https://weavz.io), the stateful agent runtime for SaaS and AI products.

The CLI signs in with an OAuth device-code flow, stores a scoped MCP OAuth token locally, and calls the hosted Weavz MCP connector at `/mcp/weavz`. It is built for local terminals, remote SSH sessions, containers, cloud IDEs, and agent runtimes that need a stable command surface.

## Links

- [Weavz](https://weavz.io)
- [Dashboard](https://dashboard.weavz.io)
- [Documentation](https://weavz.io/docs)
- [API reference](https://weavz.io/docs/api-reference)
- [Integration catalog](https://weavz.io/integrations)

## Installation

Run without a persistent install:

```bash
npx -y @weavz-io/cli login
```

Install globally when you want `weavz` available as a normal shell command:

```bash
npm install -g @weavz-io/cli
weavz --version
weavz login
```

Project-local installs do not add `weavz` to your interactive shell `PATH`; they create a project-local binary under `node_modules/.bin`. Use one of these forms inside that project:

```bash
npm install @weavz-io/cli
npx weavz --help
npm exec weavz -- actions list --json
./node_modules/.bin/weavz --help
```

If `npm install -g @weavz-io/cli` succeeds but `weavz` still says `command not found`, your npm global bin directory is not on `PATH`. On zsh, check and fix it with:

```bash
NPM_PREFIX="$(npm prefix -g)"
ls -l "$NPM_PREFIX/bin/weavz"
printf '\nexport PATH="%s/bin:$PATH"\n' "$NPM_PREFIX" >> ~/.zshrc
exec zsh -l
weavz --version
```

With `nvm`, global packages are installed per Node.js version. If you switch Node versions and `weavz` disappears, reinstall it for the active version or run it with `npx -y @weavz-io/cli`.

The package requires Node.js 22.13 or newer.

## Login

```bash
weavz login
```

The CLI prints a short code and a browser URL:

```text
First, copy this code: WVZ-ABCD-EFGH
Then open: https://platform.weavz.io/oauth/device?user_code=WVZ-ABCD-EFGH
Waiting for approval...
```

Open the URL in any browser, sign in to Weavz, choose or create a workspace, verify the code, and approve the CLI. This works even when the CLI runs inside SSH, a container, or a cloud IDE because the browser does not need to run on the same machine.

On a local interactive terminal, `weavz login` tries to open the browser automatically. It does not open the browser by default in SSH sessions, containers, cloud IDEs, CI, or non-interactive shells. Use `--no-open` when you only want the URL printed:

```bash
weavz login --no-open
```

If the environment detector is too conservative, force a browser open attempt:

```bash
weavz login --open
```

Use a custom host for development or self-hosted deployments:

```bash
weavz login --host https://platform.weavz.io
weavz login --host http://localhost:3000 --profile local
```

Credentials are stored per profile with owner-only file permissions. The config directory is selected in this order: `WEAVZ_CONFIG_DIR`, `$XDG_CONFIG_HOME/weavz`, then `~/.weavz`. Stored tokens are MCP OAuth tokens scoped to the Weavz MCP connector; they do not grant access to public REST APIs. Public REST APIs still require a `WEAVZ_API_KEY` with the `wvz_` prefix.

## Quick Start

Sign in, then let the CLI summarize the current workspace for an agent or a human:

```bash
weavz login
weavz agent context --json
```

Discover the action you need:

```bash
weavz actions search "send gmail email" --json
```

Inspect the exact input contract for one canonical action id:

```bash
weavz actions info customer_gmail.send_email --json
```

Validate input before side effects:

```bash
weavz actions validate customer_gmail.send_email \
  --set to=person@example.com \
  --set subject="Hello from Weavz" \
  --set body="Hi, this was sent through Weavz." \
  --json
```

Execute the action:

```bash
weavz run customer_gmail.send_email \
  --set to=person@example.com \
  --set subject="Hello from Weavz" \
  --set body="Hi, this was sent through Weavz." \
  --idempotency-key email-demo-001 \
  --json
```

Use Code Mode JavaScript only when a workflow needs multiple app calls, loops, joins, or transformations:

```bash
weavz exec 'return { ok: true, now: new Date().toISOString() }'
```

Pipe a larger multi-step script:

```bash
cat <<'JS' | weavz exec --json
const results = await weavz.team_slack.search_messages({
  query: "from:me deploy"
})

return results
JS
```

## Commands

### `weavz login`

Start OAuth device login.

```bash
weavz login [--host <url>] [--profile <name>] [--open] [--no-open]
```

Options:

| Option | Default | Description |
| --- | --- | --- |
| `--host <url>` | `https://platform.weavz.io` | Weavz API host |
| `--profile <name>` | `default` | Local credential profile |
| `--open` | auto | Force a browser open attempt |
| `--no-open` | - | Print the browser URL without trying to open it |

Environment:

| Variable | Description |
| --- | --- |
| `WEAVZ_HOST` | Default host for commands |
| `WEAVZ_PROFILE` | Default profile name |
| `WEAVZ_CONFIG_DIR` | Override config directory; useful in tests and containers |

### `weavz logout`

Remove a local credential profile.

```bash
weavz logout [--profile <name>]
```

### `weavz status` / `weavz whoami`

Show profile, host, workspace, MCP server, app count, tool count, and pending approval count.

```bash
weavz status
weavz whoami --json
```

### `weavz agent`

Print a bounded, agent-ready workspace snapshot or an installable instruction file:

```bash
weavz agent context [--query <intent>] [--limit <n>] [--json]
weavz agent skill [--format codex|claude|generic]
```

`agent context` includes the active profile, workspace, connected apps, matching canonical action ids, the required discovery flow, and recovery commands. It is designed so agents can bootstrap without reading this whole README.

### `weavz apps`

Manage apps attached to the connected Weavz workspace.

```bash
weavz apps list [--json]
weavz apps search <query> [--json]
weavz apps add <integration-name> [--alias <alias>] [--json]
weavz apps connect <alias> [--json]
weavz apps disconnect <alias> --confirm [--json]
weavz apps remove <alias> --confirm [--keep-connection] [--json]
weavz apps dashboard [--json]
```

Aliases are the stable names agents use in Code Mode, such as `team_slack`, `customer_gmail`, or `billing_stripe`.

### `weavz actions`

Discover, inspect, validate, and execute one canonical action at a time:

```bash
weavz actions list [--limit <n>] [--json]
weavz actions search <query> [--limit <n>] [--json]
weavz actions info <alias.action> [--json]
weavz actions options <alias.action> <property> [--search <text>] [--input <json|@file|->] [--json]
weavz actions validate <alias.action> [--input <json|@file|->] [--set <path=value>] [--set-json <path=json>] [--json]
weavz actions exec <alias.action> [input flags] [--dry-run] [--explain] [--idempotency-key <key>] [--timeout <seconds>] [--wait-for-approval-seconds <0-60>] [--json]
weavz run <alias.action> [input flags] [--dry-run] [--explain] [--idempotency-key <key>] [--timeout <seconds>] [--wait-for-approval-seconds <0-60>] [--json]
```

Canonical ids are `alias.functionName`, where `functionName` is the safe action name returned by `actions search`. Search can use natural language; execution never does. If a query returns several matches, pick one id and inspect it with `actions info`.

Input precedence is:

1. Base JSON from `--input`.
2. Repeatable scalar updates from `--set path=value`.
3. Repeatable JSON updates from `--set-json path=json`.

Input sources:

```bash
weavz run customer_gmail.send_email --input '{"to":"person@example.com","subject":"Hi","body":"Hello"}'
weavz run customer_gmail.send_email --input @email.json --json
cat email.json | weavz run customer_gmail.send_email --input - --json
weavz run team_slack.send_channel_message --set channel=C123 --set text="Deploy complete" --json
weavz run sheets.add_row --set-json 'values=["Acme",42,true]' --json
weavz run crm.create_contact --set address.city=Berlin --set tags[]=lead --set tags[]=trial --json
```

Dry-run and explain are read-only:

```bash
weavz actions exec customer_gmail.send_email --input @email.json --dry-run --json
weavz actions exec customer_gmail.send_email --input @email.json --explain
```

Resolve dynamic options before validation when a field advertises an options lookup:

```bash
weavz actions options team_slack.send_channel_message channel --search eng --json
```

`--idempotency-key` is used for approval request matching and retry coordination. Do not blindly retry side-effecting actions after an ambiguous network failure unless the response clearly returned `approval_required` or the CLI completed an auth refresh retry.

If an action requires approval, the JSON envelope has `status: "approval_required"`, an `approval` object, links, and a `requestId` when available. Show the approval URL, then retry the same command with the same input and idempotency key after approval. To keep polling briefly:

```bash
weavz run customer_gmail.send_email --input @email.json --idempotency-key email-001 --wait-for-approval-seconds 30 --json
```

Direct action JSON output is always an envelope with stable keys: `status`, `id`, `alias`, `functionName`, `actionName`, `integrationName`, `inputKeys`, `output`, `approval`, `links`, `timings`, `requestId`, and `error`.

### `weavz tools`

Discover and inspect the MCP tools exposed by the connected workspace.

```bash
weavz tools list [--json]
weavz tools search [query] [--json]
weavz tools api <alias> [alias...] [--action <action>] [--actions a,b] [--detail full|signatures] [--json]
```

Use `tools search` first when you do not know which aliases or actions are available. Use `tools api` before writing Code Mode scripts against unfamiliar aliases.

### `weavz exec`

Run JavaScript through Weavz Code Mode.

```bash
weavz exec 'return await weavz.storage.list({ path: "" })'
weavz exec --json < script.js
weavz exec --approval-id apr_123 --wait-for-approval-seconds 30 --json
```

Top-level `await` and `return` are supported by Weavz Code Mode. For workflows that involve several app calls, prefer one `weavz exec` script that fetches, transforms, and returns the result instead of many small calls.

## Agent Usage

Agents should use the direct action path first:

1. Run `weavz agent context --query "<task intent>" --json` to confirm workspace and get matching canonical ids.
2. Run `weavz actions search "<task intent>" --json` if more discovery is needed.
3. Run `weavz actions info <alias.action> --json`.
4. Build input as JSON, then run `weavz actions validate <alias.action> --input @input.json --json`.
5. Execute with `weavz run <alias.action> --input @input.json --idempotency-key <stable-key> --json`.
6. Use `weavz exec` only when the task genuinely needs a multi-step JavaScript workflow.

For approvals, keep the same idempotency key when retrying after the user approves. For ambiguous network failures after execution starts, inspect status before retrying side-effecting commands.

Example agent flow:

```bash
weavz actions search "send slack message" --json
weavz actions info support_slack.send_channel_message --json
weavz actions validate support_slack.send_channel_message \
  --set channel=C123456 \
  --set text="The build is green." \
  --json
weavz run support_slack.send_channel_message \
  --set channel=C123456 \
  --set text="The build is green." \
  --idempotency-key build-green-20260617 \
  --json
```

Code Mode remains useful for multi-action workflows:

```bash
weavz exec --json <<'JS'
const messages = await weavz.support_slack.search_messages({
  query: "escalation after:2026-06-01"
})

return {
  count: messages.length,
  summaries: messages.slice(0, 10).map(message => ({
    text: message.text,
    user: message.user,
    ts: message.ts
  }))
}
JS
```

## Authentication Model

`weavz login` uses OAuth 2.0 Device Authorization Grant:

- The CLI requests a device code from `POST /oauth/device/code`.
- The user approves the request in a browser at `/oauth/device`.
- The CLI polls `POST /oauth/token` with `grant_type=urn:ietf:params:oauth:grant-type:device_code`.
- The server returns an MCP OAuth access token and refresh token.
- The CLI refreshes before use and retries once after a 401 from `/mcp/weavz`.

The OAuth token is scoped to MCP connector usage. It is not a Weavz REST API key.

## Security Notes

- Do not commit `~/.weavz/credentials.json`.
- Use separate profiles for development, staging, and production.
- Use `--host http://localhost:3000 --profile local` for local development against a self-hosted Weavz API.
- For CI automation, prefer purpose-scoped Weavz API keys where possible. The CLI login flow is designed for interactive users and agents acting with user approval.
- If a credential is exposed, run `weavz logout --profile <name>` locally and revoke/reconnect the MCP connector from Weavz.

## Troubleshooting

### Login hangs at "Waiting for approval"

Open the printed `verification_uri_complete` URL and verify the code shown in the browser matches the terminal code. Device codes expire after 15 minutes.

### The browser does not open

Use the printed URL. This is expected in SSH sessions, containers, cloud IDEs, CI, and non-interactive shells. On a local machine, retry with `--open` to force a browser open attempt:

```bash
weavz login --open
```

When you intentionally want URL-only login:

```bash
weavz login --no-open
```

### `Not logged in for profile`

Run login for that profile:

```bash
weavz login --profile my-profile
```

Or set the profile:

```bash
export WEAVZ_PROFILE=my-profile
```

### REST API requests fail with the CLI token

This is expected. CLI tokens are MCP OAuth tokens for `/mcp/weavz`. Use the SDKs or REST API with `WEAVZ_API_KEY` for public API routes.

## Issues

Report CLI issues in the [weavz-cli repository](https://github.com/weavz/weavz-cli/issues). Include `weavz --version`, your operating system, whether you are running locally or over SSH/container/cloud IDE, and the command output with secrets removed.

## License

MIT
