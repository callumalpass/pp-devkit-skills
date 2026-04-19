---
name: pp
description: Use pp, the Microsoft Power Platform CLI/MCP/UI, for account auth, environment management, Dataverse, Power Automate, Microsoft Graph, SharePoint REST, BAP, Power Apps, Canvas Authoring, jq-filtered API requests, and Power Platform investigation workflows. Trigger when the user mentions pp, Power Platform APIs, Dataverse, Power Automate flows, SharePoint REST, Canvas apps, Power Apps authoring, or asks to query/inspect/mutate a configured environment.
---

# pp

`pp` is a Node 22+ CLI, library, MCP server, and localhost UI for Microsoft
Power Platform work. It provides authenticated requests against Dataverse, Power
Automate, Microsoft Graph, SharePoint REST, BAP, Power Apps, and the internal
Canvas Authoring service.

Source repo on this machine: `/home/calluma/projects/pp`.

## First Choice

If pp MCP tools are available in the current assistant, prefer them for account,
environment, and API requests. They avoid shell quoting problems and support the
same request options as the CLI, including `jq`, `readIntent`, `rawBody`,
`responseType`, and `allowInteractiveAuth`.

Use the CLI when the user asks for terminal commands, when MCP tools are not
available, or for commands not exposed by MCP such as `pp ui` and
`pp canvas-authoring yaml fetch --out`.

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
pp request [dv|flow|graph|sharepoint|bap|powerapps|canvas-authoring|custom] <path|url> [--env <alias>|--account <account>]
pp <api> <path|url> [--env <alias>|--account <account>]
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
- `--via-ui`
- `--temp-token NAME`
- `--no-interactive-auth`

`--jq` runs in-process with jq-wasm; use API-native `$select`, `$filter`, and
`$top` first, then use `--jq` to reshape returned JSON.

Absolute URLs are allowed for all APIs. If no API is specified, pp auto-detects
Graph, SharePoint, Power Apps, Canvas Authoring, BAP, Flow, Dataverse API URLs,
or falls back to `custom` for other absolute URLs and `dv` for relative paths.

Dataverse, Flow, BAP, Power Apps, Canvas Authoring, and custom requests are
environment-scoped and require `--env`. Graph and SharePoint are account-scoped:
use `--account` directly, or pass `--env` as a shorthand for the environment's
configured account.

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

`pp flow` is a Flow API request shortcut. It does not wrap workflow-definition
inspection or validation behind custom CLI verbs; agents should construct the
canonical Flow API requests directly.

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

For canonical Power Automate definition validation, call the Flow service
validation endpoints with the candidate flow payload. Use the existing flow id
when validating an update; for pre-create checks, the all-zero flow id has been
observed to work with the same request shape.

```sh
pp flow /flows/<flow-id>/checkFlowErrors --env <alias> --method POST --body-file payload.json
pp flow /flows/<flow-id>/checkFlowWarnings --env <alias> --method POST --body-file payload.json
pp flow /flows/00000000-0000-0000-0000-000000000000/checkFlowErrors --env <alias> --method POST --body-file payload.json
pp flow /flows/00000000-0000-0000-0000-000000000000/checkFlowWarnings --env <alias> --method POST --body-file payload.json
```

The validation body should use the Power Automate service shape, not a bare
workflow definition: `{"properties":{"definition":...,"connectionReferences":{},"displayName":...}}`.

For connector and operation metadata, prefer the catalog endpoints rather than
hard-coded connector schemas:

```sh
pp flow /operationGroups --env <alias> --method POST --body '{"searchText":"","visibleHideKeys":[],"usage":"Action","allTagsToInclude":[],"anyTagsToExclude":["ToS","Agentic"]}'
pp flow /operations --env <alias> --method POST --query '$top=250' --body '{"searchText":"","visibleHideKeys":[],"allTagsToInclude":["Action","Important"],"anyTagsToExclude":["Deprecated","Agentic","Trigger"]}'
pp flow /operationGroups/<group>/operations/<operation> --env <alias>
pp flow /apis/<connector>/apiOperations/<operation> --env <alias>
pp flow /apis/<connector> --env <alias>
```

### Creating And Activating Solution Cloud Flows

For solution cloud flows, prefer the supported Dataverse `workflow` table over
raw `api.flow.microsoft.com` `/flows` creation. Microsoft documents
`api.flow.microsoft.com` as unsupported for code-based solution flow management;
use Dataverse Web API or the management connectors instead.

Create the flow with Dataverse:

```sh
pp dv /workflows --env <alias> --method POST --header 'Prefer:return=representation' --body-file create-flow.json
```

Minimum create payload:

```json
{
  "category": 5,
  "name": "My scheduled Dataverse probe",
  "type": 1,
  "primaryentity": "none",
  "clientdata": "{\"properties\":{\"connectionReferences\":{},\"definition\":{}},\"schemaVersion\":\"1.0.0.0\"}"
}
```

`category: 5` means Modern Flow, `type: 1` means Definition, and
`primaryentity: "none"` is used for automated, instant, and scheduled cloud
flows. `clientdata` is a JSON string containing `properties.definition` and
`properties.connectionReferences`.

For connector-backed actions in solution flows, the connection reference should
normally point at a Dataverse connection reference logical name, not at the
concrete API Hub connection name. A Dataverse action reference should look like:

```json
{
  "runtimeSource": "embedded",
  "connection": {
    "connectionReferenceLogicalName": "new_sharedcommondataserviceforapps_7f348"
  },
  "api": {
    "name": "shared_commondataserviceforapps"
  }
}
```

Do not include `connection.name` in the stored `clientdata` unless you have
evidence the target endpoint expects it. In live testing, flows created with the
concrete API Hub connection name could validate and appear installed, but failed
activation with `InvalidOpenApiFlow` or Flow `/start` returned
`CannotStartUnpublishedSolutionFlow`.

The workflow definition must include the normal `$connections` and
`$authentication` parameters, and connector actions should reference the
connection-reference key through `inputs.host.connectionName`:

```json
{
  "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "$connections": { "defaultValue": {}, "type": "Object" },
    "$authentication": { "defaultValue": {}, "type": "SecureObject" }
  },
  "triggers": {
    "Recurrence": {
      "recurrence": { "frequency": "Day", "interval": 1 },
      "metadata": { "operationMetadataId": "<uuid>" },
      "type": "Recurrence"
    }
  },
  "actions": {
    "List_accounts": {
      "runAfter": {},
      "metadata": { "operationMetadataId": "<uuid>" },
      "type": "OpenApiConnection",
      "inputs": {
        "host": {
          "connectionName": "shared_commondataserviceforapps",
          "operationId": "ListRecords",
          "apiId": "/providers/Microsoft.PowerApps/apis/shared_commondataserviceforapps"
        },
        "parameters": { "entityName": "accounts", "$top": 1 },
        "authentication": "@parameters('$authentication')"
      }
    }
  }
}
```

Activate with Dataverse, not Flow `/start`:

```sh
pp dv '/workflows(<workflowid>)' --env <alias> --method PATCH --body '{"statecode":1}' --response-type void
```

After activation, verify through the Flow API:

```sh
pp flow /flows/<workflowid> --env <alias> --jq '{name, resourceId:.properties.resourceId, state:.properties.state, installationStatus:.properties.installationStatus, installed:.properties.installedConnectionReferences}'
```

A successfully activated connector-backed flow should report `state: "Started"`,
`installationStatus: "Installed"`, a non-empty `resourceId`, and
`installedConnectionReferences.*.impersonation.objectId`. If `resourceId` is
set, runtime operations such as trigger `run` and run history may need the
runtime id rather than the Dataverse `workflowid`:

```sh
pp flow /flows/<resourceId>/triggers/Recurrence/run --env <alias> --method POST --response-type void
pp flow '/flows/<resourceId>/runs?$top=5' --env <alias>
```

`PvaShareConnection` is an internal Dataverse action on `connectionreference`;
do not rely on it for ordinary flow activation. If it returns a permission error
for the same caller that owns the API Hub connection, treat that as a sign the
wrong activation path is being used rather than as a missing share to fix.

For run investigations, inspect both `/runs` and trigger histories. Polling
triggers may show useful state in
`/flows/<flow-id>/triggers/<trigger-name>/histories` even when no parent run was
created. Action `inputsLink` and `outputsLink` URLs are pre-signed; fetch them
directly without auth headers.

## Graph, SharePoint, BAP, And Power Apps

```sh
pp graph /me --env <alias>
pp graph /me --account <account>
pp graph /users --env <alias> --query '$top=5'

pp sp https://contoso.sharepoint.com/sites/site/_api/web --account <account>
pp sharepoint https://contoso.sharepoint.com/sites/site/_api/web/lists --env <alias>

pp bap /environments --env <alias>
pp bap /environments/<maker-environment-id> --env <alias>

pp powerapps /apps --env <alias>
pp powerapps /apps/<app-id> --env <alias>
pp powerapps /connections --env <alias>
```

Graph relative paths are rooted at `/v1.0` unless the path starts with
`/v1.0/` or `/beta/`. SharePoint REST requests require a full SharePoint URL.
The SharePoint token audience is the URL origin, such as
`https://contoso.sharepoint.com` for
`https://contoso.sharepoint.com/sites/site/_api/web`. BAP paths are rooted at
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

### Temporary Access Tokens

The UI can hold short-lived pasted browser bearer tokens for APIs that `pp`
cannot acquire directly, such as some SharePoint resources. In the UI, go to
`Setup -> Advanced -> Temporary Access Tokens`, paste the bearer token, and
choose how it should match requests:

- Infer from token audience
- URL origin, such as `https://contoso.sharepoint.com`
- pp API, such as `graph`
- Token audience

The token stays only in the running UI server process. It is not written to
`config.json`, and it disappears when the UI server exits or the user clicks
`Forget`. The UI shows decoded JWT metadata such as `aud`, subject, scopes,
roles, and expiry, but never shows the token value again.

To use a UI-held token from the CLI, route the request through the running UI
server:

```sh
pp request --via-ui --temp-token sharepoint custom https://contoso.sharepoint.com/sites/site/_api/web --env <alias>
```

`--via-ui` reads the UI state file, authenticates to the localhost UI server with
the per-session CLI secret, and asks the UI server to execute the request. Use
`--temp-token NAME` when possible so the intended token is explicit. Without
`--temp-token`, the UI server may auto-match a non-expired temporary token by
the request API/origin.

The URL origin is the scheme plus host plus optional port only. For
`https://contoso.sharepoint.com/sites/foo/_api/web`, the origin is
`https://contoso.sharepoint.com`.

Prefer this flow for browser-only access gaps. Prefer normal `pp auth login`,
`--env-token`, or `--static-token` when a reusable CLI account is actually
desired.

## MCP Server

```sh
pp mcp
pp mcp --allow-interactive-auth
pp-mcp --tool-name-style underscore
```

Default MCP tool names are dotted:

- `pp.account.list`, `pp.account.inspect`, `pp.account.login`, `pp.account.remove`
- `pp.environment.list`, `pp.environment.inspect`, `pp.environment.add`, `pp.environment.discover`, `pp.environment.remove`
- `pp.request`, `pp.dv_request`, `pp.flow_request`, `pp.graph_request`, `pp.sharepoint_request`, `pp.bap_request`, `pp.powerapps_request`
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
- Graph and SharePoint are account-scoped and can use `--account` without
  `--env`; environment-scoped APIs still require `--env`.
- `pp sharepoint` and `pp sp` require full SharePoint REST URLs and authenticate
  to the SharePoint origin.
- Flow, BAP, and Power Apps add default `api-version` query values
  automatically. Passing the same query key overrides the default.
- For read-only environment aliases, pass `--read` only when a POST is known to
  be semantically read-only.
- Prefer `--response-type text` for non-JSON responses and `--response-type
  void` for deletes, trigger calls, or endpoints where the body is irrelevant.
- `--body` parses JSON; use `--raw-body` for XML, text, or already encoded
  payloads.
- UI temporary access tokens require a running `pp ui` instance and CLI requests
  must opt in with `--via-ui`. Treat pasted tokens like passwords: never ask the
  user to paste one into chat, never log one, and prefer matching by exact URL
  origin for SharePoint.

## Keeping This Skill Current

When pp changes, update this skill before ending the session if the change
affects user-facing commands, flags, auth behavior, MCP tools, Canvas Authoring
workflow, or common failure modes. Keep local customer/environment names out of
the skill unless the user explicitly asks for a personal operating note; prefer
`pp env list` and `pp auth list` for live state.
