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

Run without installing:

```bash
npx -y @weavz-io/cli login
```

Install globally:

```bash
npm install -g @weavz-io/cli
weavz login
```

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

On a local interactive terminal, `weavz login` may try to open the browser automatically. Use `--no-open` when you only want the URL printed:

```bash
weavz login --no-open
```

Use a custom host for development or self-hosted deployments:

```bash
weavz login --host https://platform.weavz.io
weavz login --host http://localhost:3000 --profile local
```

Credentials are stored per profile with owner-only file permissions. The config directory is selected in this order: `WEAVZ_CONFIG_DIR`, `$XDG_CONFIG_HOME/weavz`, then `~/.weavz`. Stored tokens are MCP OAuth tokens scoped to the Weavz MCP connector; they do not grant access to public REST APIs. Public REST APIs still require a `WEAVZ_API_KEY` with the `wvz_` prefix.

## Quick Start

Check the active profile:

```bash
weavz status
weavz whoami --json
```

List configured apps:

```bash
weavz apps list
```

Search the app catalog:

```bash
weavz apps search slack
weavz apps search "google sheets"
```

Add an app to the connected workspace:

```bash
weavz apps add slack --alias team_slack
```

Connect your user account for an app alias:

```bash
weavz apps connect team_slack
```

Search available agent tools:

```bash
weavz tools search "slack gmail storage"
```

Inspect TypeScript declarations for an alias:

```bash
weavz tools api team_slack
```

Execute Code Mode JavaScript:

```bash
weavz exec 'return { ok: true, now: new Date().toISOString() }'
```

Pipe a larger script:

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
weavz login [--host <url>] [--profile <name>] [--no-open]
```

Options:

| Option | Default | Description |
| --- | --- | --- |
| `--host <url>` | `https://platform.weavz.io` | Weavz API host |
| `--profile <name>` | `default` | Local credential profile |
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

Agents should treat the CLI as a narrow MCP connector command surface:

1. Run `weavz status --json` to confirm the active profile and workspace.
2. Run `weavz tools search "<task intent>" --json` to discover relevant aliases and actions.
3. Run `weavz tools api <alias> --json` for declarations before calling unfamiliar actions.
4. Run one `weavz exec` script for the workflow and return structured JSON.
5. If `weavz exec` reports an approval requirement, surface the approval link or instruction to the user, then continue with `weavz exec --approval-id <apr_...> --wait-for-approval-seconds 30 --json`.

Example agent flow:

```bash
weavz tools search "read support slack summarize recent escalations" --json
weavz tools api support_slack --json
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

Use the printed URL. This is expected in SSH sessions, containers, and many cloud IDEs.

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
