---
name: pp
description: Use pp, the Microsoft Power Platform CLI/MCP/UI, for account auth, environment management, Dataverse, Power Automate, Microsoft Graph, BAP, Power Apps, Canvas Authoring, jq-filtered API requests, and Power Platform investigation workflows. Trigger when the user mentions pp, Power Platform APIs, Dataverse, Power Automate flows, Canvas apps, Power Apps authoring, or asks to query/inspect/mutate a configured environment.
---

# pp

`pp` is a Node 22+ CLI, library, MCP server, and localhost UI for Microsoft
Power Platform work. It provides authenticated requests against Dataverse, Power
Automate, Microsoft Graph, BAP, Power Apps, and the internal Canvas Authoring
service.

Source repo on this machine: `/home/calluma/projects/pp`.

## First Choice

If pp MCP tools are available in the current assistant, prefer them for account,
environment, and API requests. They avoid shell quoting problems and support the
same request options as the CLI, including `jq`, `readIntent`, `rawBody`,
`responseType`, and `allowInteractiveAuth`.

Use the CLI when the user asks for terminal commands, when MCP tools are not
available, or for commands not exposed by MCP such as `pp ui`, `pp flow
validate`, and `pp canvas-authoring yaml fetch --out`.

## Invocation

```sh
pp --version
npx pp --help
node /home/calluma/projects/pp/dist/index.cjs --help
```

The npm package exposes:

- `pp` - full CLI
- `pp-mcp` - MCP server entry point
- `pp-ui` - browser UI launcher

From source:

```sh
cd /home/calluma/projects/pp
pnpm install
pnpm build
```

## Config Model

`pp` stores accounts, environments, UI state, MSAL cache, browser profiles, and
Canvas sessions in a global config directory.

- Linux/macOS default: `$XDG_CONFIG_HOME/pp` or `~/.config/pp`
- Windows default: `%APPDATA%\pp`
- Override any command with `--config-dir DIR`
- Main config file: `config.json`

Do not assume a repo-local `pp.config.yaml`; migrate old configs with
`pp migrate-config [--source-config PATH|--source-dir DIR] --apply`.

Use `pp auth list` and `pp env list` for current state instead of hard-coding
account or environment names in this skill.

## Accounts

```sh
pp auth list
pp auth inspect <account>
pp auth login <account>
pp auth login <account> --device-code
pp auth login <account> --client-secret --tenant-id <tenant> --client-id <client> --client-secret-env SECRET_ENV_VAR
pp auth login <account> --env-token --env-var ACCESS_TOKEN_ENV_VAR
pp auth login <account> --static-token --token <token>
pp auth remove <account>
```

Useful login flags:

- `--login-hint USER`
- `--prompt select_account|login|consent|none`
- `--device-code-fallback`
- `--force-prompt`
- `--no-interactive-auth`

When browser auth hangs or the tenant blocks the localhost flow, retry with
`--device-code`. If a noninteractive request fails because a token expired,
rerun without `--no-interactive-auth` or refresh with `pp auth login`.

## Environments

```sh
pp env list
pp env inspect <alias>
pp env discover <account>
pp env add <alias> --url https://org.crm.dynamics.com --account <account>
pp env add <alias> --url https://org.crm.dynamics.com --account <account> --access read-only
pp env remove <alias>
```

`pp env add` discovers and stores the Maker environment id and tenant id. Most
runtime commands take `--env <alias>` and optionally `--account <account>` to
override the environment's default account.

Read-only environments block non-GET/HEAD requests unless `--read` is passed.
Use `--read` only for POST endpoints that are known to be read-only, such as
some metadata, WhoAmI, or authoring read endpoints.

## Connectivity

```sh
pp whoami --env <alias>
pp ping --env <alias> --api dv|flow|graph|bap|powerapps|canvas-authoring
pp token --env <alias> --api dv|flow|graph|bap|powerapps|canvas-authoring
pp token --env <alias> --api canvas-authoring --device-code
```

`pp whoami` uses the Dataverse WhoAmI operation. If it is not enough for a
tenant-specific issue, make an explicit Dataverse request:

```sh
pp dv /WhoAmI --env <alias> --method GET
```

## Request Commands

General form:

```sh
pp request [dv|flow|graph|bap|powerapps|canvas-authoring|custom] <path|url> --env <alias>
pp <api> <path|url> --env <alias>
```

Common flags:

- `--method GET|POST|PATCH|DELETE`
- `--query K=V` repeated
- `--header 'Name: value'` repeated
- `--body JSON` or `--body-file FILE` for JSON bodies
- `--raw-body TEXT` or `--raw-body-file FILE` for non-JSON bodies
- `--response-type json|text|void`
- `--timeout-ms MS`
- `--jq EXPR`
- `--format json|yaml|text`
- `--read`
- `--no-interactive-auth`

`--jq` runs in-process with jq-wasm; use API-native `$select`, `$filter`, and
`$top` first, then use `--jq` to reshape returned JSON.

Absolute URLs are allowed for all APIs. If no API is specified, pp auto-detects
Graph, Power Apps, Canvas Authoring, BAP, Flow, Dataverse API URLs, or falls
back to `custom` for other absolute URLs and `dv` for relative paths.

## Dataverse

`pp dv` roots relative paths at `/api/data/v9.2` and authenticates against the
environment URL.

```sh
pp dv /accounts --env <alias> --query '$select=name,accountid' --query '$top=10'
pp dv /accounts --env <alias> --query '$filter=statecode eq 0'
pp dv /accounts(<guid>) --env <alias>
pp dv /accounts --env <alias> --method POST --body '{"name":"Test"}'
pp dv /accounts(<guid>) --env <alias> --method PATCH --body '{"name":"Updated"}'
pp dv /accounts(<guid>) --env <alias> --method DELETE --response-type void
pp dv /EntityDefinitions --env <alias> --query '$select=LogicalName,EntitySetName'
```

PowerShell, zsh, and bash quote `$select`, `$filter`, and `$top` differently.
Prefer single quotes around OData query arguments in POSIX shells. In Git Bash
on Windows, prefix commands with `MSYS_NO_PATHCONV=1` when paths beginning with
`/` are being rewritten.

## Power Automate

`pp flow` is both a Flow API shortcut and a workflow-definition analyzer.

API examples:

```sh
pp flow /flows --env <alias>
pp flow /flows/<flow-id> --env <alias>
pp flow /flows/<flow-id>/runs --env <alias>
pp flow /flows/<flow-id>/runs/<run-id>/actions/<action-name> --env <alias>
pp flow /flows/<flow-id>/triggers/manual/listCallbackUrl --env <alias> --method POST
```

Relative Flow paths are rooted at
`/providers/Microsoft.ProcessSimple/environments/<makerEnvironmentId>`.
`api-version=2016-11-01` is added automatically.

For non-standard Flow URLs under `https://api.flow.microsoft.com/providers/...`,
pass the full URL with `pp flow <url>` or `pp request flow <url>` so pp keeps
the Flow auth resource but does not prepend the environment path.

Flow definition helpers:

```sh
pp flow validate workflow.json
pp flow inspect workflow.json
pp flow symbols workflow.json
pp flow explain workflow.json --symbol <action-or-trigger-name>
```

For run investigations, inspect both `/runs` and trigger histories. Polling
triggers may show useful state in
`/flows/<flow-id>/triggers/<trigger-name>/histories` even when no parent run was
created. Action `inputsLink` and `outputsLink` URLs are pre-signed; fetch them
directly without auth headers.

## Graph, BAP, And Power Apps

```sh
pp graph /me --env <alias>
pp graph /users --env <alias> --query '$top=5'

pp bap /environments --env <alias>
pp bap /environments/<maker-environment-id> --env <alias>

pp powerapps /apps --env <alias>
pp powerapps /apps/<app-id> --env <alias>
pp powerapps /connections --env <alias>
```

Graph relative paths are rooted at `/v1.0` unless the path starts with
`/v1.0/` or `/beta/`. BAP paths are rooted at
`/providers/Microsoft.BusinessAppPlatform` with default
`api-version=2020-10-01`. Power Apps paths are rooted at
`/providers/Microsoft.PowerApps` with default `api-version=2016-11-01`; the
literal `{environment}` in a path is replaced with the Maker environment id.

## Canvas Authoring

`pp canvas-authoring` targets the internal Power Apps Studio authoring service.
These APIs are stateful, versioned, and not a public contract. Treat mutation
commands carefully because valid YAML/RPC calls can update the dirty draft open
in Maker/Studio.

Start with a live session:

```sh
pp canvas-authoring /gateway/cluster --env <alias> --read
pp canvas-authoring session start --env <alias> --app <app-id>
pp canvas-authoring session list
pp canvas-authoring session request --env <alias> --app <app-id> --path /api/yaml/fetch --read
```

YAML and metadata helpers:

```sh
pp canvas-authoring yaml fetch --env <alias> --app <app-id> --out ./canvas-src
pp canvas-authoring yaml validate --env <alias> --app <app-id> --dir ./canvas-src
pp canvas-authoring yaml validate --env <alias> --app <app-id> --dir ./canvas-src --with-signalr
pp canvas-authoring controls list --env <alias> --app <app-id>
pp canvas-authoring controls describe --env <alias> --app <app-id> Label
pp canvas-authoring apis list --env <alias> --app <app-id>
pp canvas-authoring datasources list --env <alias> --app <app-id>
pp canvas-authoring accessibility --env <alias> --app <app-id>
```

Low-level document-server calls:

```sh
pp canvas-authoring invoke --env <alias> --app <app-id> --class documentservicev2 --oid 1 --method keepalive
pp canvas-authoring rpc --env <alias> --app <app-id> --class document --oid 2 --method geterrorsasync
```

Use `rpc` for query-style methods that return a document-server result over the
authoring SignalR channel. Use `invoke` for direct `/api/v2/invoke` calls where
HTTP success is sufficient. Object ids such as `document/2` come from the live
session and are not stable across all contexts.

Canvas Authoring uses a first-party client and a separate token cache for user
and device-code accounts. Device code is normal here because the Studio client
does not support pp's localhost browser callback.

Known limitation: the YAML round trip is unreliable for apps that use Canvas
components or component libraries. Use `yaml fetch`/`validate` primarily for
apps and screens without components, and require clean diagnostics plus
`hasActiveCoauthoringSession: true` before trusting a validate result.

## Custom Requests

Use `custom` only for arbitrary full URLs such as webhook URLs or Flow HTTP
trigger callback URLs:

```sh
pp request custom https://prod-xx.logic.azure.com/workflows/... --env <alias> --method POST --body '{}' --response-type void
```

The auth resource for `custom` is the URL origin. For anonymous pre-signed URLs,
`curl` may be simpler.

## UI

```sh
pp ui
pp ui --port 4734
pp ui --no-open
pp ui --lan --pair --no-open
```

The UI manages accounts/environments, checks setup, explores Dataverse metadata,
runs OData and FetchXML queries, and tracks long-running jobs. `pp ui` reuses an
existing server for the same config dir when possible and falls back to another
localhost port if needed.

LAN mode requires `--pair`; use it only on a trusted network.

## MCP Server

```sh
pp mcp
pp mcp --allow-interactive-auth
pp-mcp --tool-name-style underscore
```

Default MCP tool names are dotted:

- `pp.account.list`, `pp.account.inspect`, `pp.account.login`, `pp.account.remove`
- `pp.environment.list`, `pp.environment.inspect`, `pp.environment.add`, `pp.environment.discover`, `pp.environment.remove`
- `pp.request`, `pp.dv_request`, `pp.flow_request`, `pp.graph_request`, `pp.bap_request`, `pp.powerapps_request`
- `pp.whoami`, `pp.ping`, `pp.token`

Use `--tool-name-style underscore` for clients that reject dotted tool names,
such as GitHub Copilot MCP integrations.

## Updates And Completion

```sh
pp update
pp version
pp completion zsh
pp completion bash
pp completion powershell
```

Most commands passively check for update notices in the background. `mcp`,
`token`, `completion`, `update`, and `version` skip passive notices.

## Gotchas

- The current source path is `/home/calluma/projects/pp`; older references to
  `power-platform-devkit` or `packages/cli/dist/index.cjs` are stale.
- `pp request` and API shortcuts now return `{ request, response, status,
  headers }`; update old scripts that read `.body` to read `.response`.
- The old `pp dv request <path>` style is gone; use `pp dv <path>`.
- Flow, BAP, and Power Apps add default `api-version` query values
  automatically. Passing the same query key overrides the default.
- For read-only environment aliases, pass `--read` only when a POST is known to
  be semantically read-only.
- Prefer `--response-type text` for non-JSON responses and `--response-type
  void` for deletes, trigger calls, or endpoints where the body is irrelevant.
- `--body` parses JSON; use `--raw-body` for XML, text, or already encoded
  payloads.

## Keeping This Skill Current

When pp changes, update this skill before ending the session if the change
affects user-facing commands, flags, auth behavior, MCP tools, Canvas Authoring
workflow, or common failure modes. Keep local customer/environment names out of
the skill unless the user explicitly asks for a personal operating note; prefer
`pp env list` and `pp auth list` for live state.
