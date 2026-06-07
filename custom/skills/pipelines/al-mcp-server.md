---
kind: task-skill
id: al-mcp-server
version: 1
title: Drive AL build and compile with the AL MCP Server
description: Use the standalone AL MCP Server (altool launchmcpserver) to drive AL build, compile, publish, symbol download, and symbol search from any MCP-compatible agent. Use when wiring AL development into CI, Copilot Agent Mode outside VS Code, or any non-VS-Code AI workflow.
bc-version: [all]
technologies: [al, powershell]
countries: [w1]
application-area: [all]
---

# AL MCP Server

The standalone Model Context Protocol server that exposes AL developer tools (build, compile, publish, symbols, diagnostics, auth) to any MCP-compatible client. Same tool names and behaviour as the VS Code Language Model Tools, but runs as its own process and works headless.

## When to use

- Wiring AL build/test into a CI pipeline that isn't VS Code
- Driving AL development from Claude, Copilot Studio, custom agents, or any other MCP client
- Headless agents that need to build, compile, publish, or search symbols
- Quick PR-gate scripts that need fast compile feedback (`al_compile` is faster than `al_build`)

## Prerequisites

- .NET 8 runtime
- AL Language extension 17.0 or later (provides the `altool` binary)
- Network access to the Business Central environment for tools that hit a server (`al_publish`, `al_downloadsymbols`)

## Starting the server

**STDIO (default for most agents):**

```bash
altool launchmcpserver --transport stdio
```

Reads JSON-RPC requests on `stdin`, writes responses on `stdout`, diagnostics on `stderr`. Shuts down on `stdin` EOF or SIGTERM.

**HTTP (for network-attached agents):**

```bash
altool launchmcpserver --transport http --port 5010
```

**Claude / generic agent config:**

```json
{
  "mcpServers": {
    "al": {
      "command": "altool",
      "args": ["launchmcpserver", "--transport", "stdio"]
    }
  }
}
```

## Tools exposed

| Tool | VS Code | AL MCP | What it does |
|---|---|---|---|
| `al_build` | yes | yes | Compile and produce a `.app` package |
| `al_compile` | no | yes | Validate AL code without packaging (faster than `al_build`, AL MCP only) |
| `al_publish` | yes | yes | Deploy `.app` to BC cloud or on-prem |
| `al_downloadsymbols` | yes | yes | Pull `.app` symbol packages to `.alpackages/` |
| `al_symbolsearch` | yes | yes | Search objects and members across project and dependencies |
| `al_getdiagnostics` | yes | yes | Read diagnostics from the last compilation |
| `al_getpackagedependencies` | no | yes | Read declared dependencies (AL MCP only) |
| `al_auth_login` | no | yes | MSAL interactive sign-in to BC cloud (AL MCP only) |
| `al_auth_logout` | no | yes | Clear cached MSAL token (AL MCP only) |

## JSON-RPC envelope

All calls follow the standard MCP shape:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "al_compile",
    "arguments": { "onlyErrors": true }
  }
}
```

**`al_symbolsearch` is the only tool whose arguments wrap under a `parameters` key.** Every other tool puts arguments at the top level under `params.arguments`.

```json
{
  "name": "al_symbolsearch",
  "arguments": {
    "parameters": {
      "query": "Post",
      "filters": { "kinds": ["Codeunit"], "scope": "project" }
    }
  }
}
```

## Workflow patterns

### CI gate (fast)

```
al_auth_login (if cloud symbols needed)
  -> al_downloadsymbols (globalSourcesOnly=true for offline)
  -> al_compile (onlyErrors=true)
  [if fail] -> al_getdiagnostics
  [if pass] -> exit 0
```

`al_compile --onlyErrors` is the right gate for PR checks: skips `.app` packaging, returns `{Succeeded, Diagnostics, Message}`. Faster than `al_build`.

### Build and deploy

```
al_downloadsymbols
  -> al_build (produces .app)
  -> al_publish (appPath, environmentName, environmentType, tenant)
```

### Headless symbol exploration

```
al_symbolsearch with query="*" and filters tuned for the question.
```

## Per-tool parameter cheat sheet

### `al_build`

- `scope`: `current` (default) or `all`
- `projectPath`: target a specific project in multi-project workspaces
- `outputPath`: where to write the `.app`
- `onlyErrors`: bool, default false
- `maxDiagnostics`: int, default 100
- `enableCodeAnalysis`: bool
- `codeAnalyzers`: array of `${CodeCop}`, `${AppSourceCop}`, `${PerTenantExtensionCop}`, `${UICop}`

The `.app` is produced only when build has zero errors. Warnings still produce a `.app`.

### `al_compile`

- `onlyErrors`: bool, default **true**
- `maxDiagnosticsPerCompilation`: int, default 100
- `enableCodeAnalysis`: bool
- `codeAnalyzers`: same analyzer placeholders as `al_build`

Returns `{Succeeded, Diagnostics[{Severity, Code, Location, Description}], Message}`. Diagnostic `Location` is `"MyCodeunit.al(42,15)"`.

### `al_publish`

Provide one of `appPath` or `projectPath`. Then add either cloud or on-prem connection params.

| Cloud | On-prem |
|---|---|
| `environmentName`, `environmentType` (`Sandbox`/`Production`), `tenant` | `serverUrl`, `serverInstance`, `port`, `authentication` (`AAD`/`Windows`/`UserPassword`, default `AAD`) |

Common options: `schemaUpdateMode` (`Synchronize`/`ForceSync`/`Recreate`, default `Synchronize`), `forceUpgrade`, `skipBuild`, `buildDependencies`, `useInteractiveLogin` (default true), `noCache`.

VS Code-only extras: `debug` (auto-attach debugger), `type` (`full`/`incremental`), `fulldependencytree`, `skipbuild`.

### `al_downloadsymbols`

Pulls symbols to `.alpackages/`. Set `globalSourcesOnly: true` for CI (no BC server connection or auth, only Microsoft NuGet feeds and AppSource). Other params: `projectPath`, `force`, `noCache`, `useInteractiveLogin` (default true), and the same cloud / on-prem overrides as `al_publish` to override `launch.json`.

### `al_symbolsearch`

Wrap arguments under `parameters`. Filters:

- `kinds`: `["Table", "Codeunit", "Page", "Report", "Enum", "Interface"]`
- `objectName`: search within a specific object
- `memberKinds`: `["Field", "Method", "Key", "Action", "Trigger"]`
- `namespace`, `access` (`["Public", "Internal"]`), `obsoleteState` (`["No", "Pending", "Removed"]`)
- `match`: `name` / `doc` / `all` (default `name`)
- `scope`: `project` / `dependencies` / `all` (default `all`)
- `limit`: max 200

Returns `symbols[]` with `{id, name, fullName, kind, namespace, containerName, signature, docSummary, path}` and `truncated: bool`.

## Authentication

AL MCP uses MSAL interactive (browser-based) auth for cloud calls. Tokens cache on disk and reuse across calls.

```
al_auth_login(tenant, environment)   # call first per session
  -> tokens cached
  -> subsequent al_publish / al_downloadsymbols reuse the cache
al_auth_logout                       # clear the cache
```

On-prem with Windows auth needs no explicit sign-in.

## Common gotchas

- **`al_symbolsearch` requires the `parameters` wrapper.** Every other tool puts args at top level. Easy to miss.
- **`al_compile` does not exist in VS Code Language Model Tools.** Use `al_build` with `scope: "current"` in VS Code for an equivalent.
- **Doc URL slug**: the page is `al-tool-symbol-search` (hyphenated), not `al-tool-symbolsearch`. Some Microsoft links use the wrong slug.
- **Single compilation session persists** across calls for the process lifetime. Calling multiple tools in sequence is faster than restarting the server.
- **Token cache** is per-user on disk. `noCache: true` forces a fresh sign-in if the cached token has the wrong scopes.

## Related skills

- `al-go-pipelines` for wiring `altool launchmcpserver` into AL-Go CI
- `troubleshooting-mcp-server` for debug-time AI assistance
- `bc-mcp-server-data` for the BC product MCP server (data, not dev tools)

## References

- AL MCP server: https://learn.microsoft.com/dynamics365/business-central/dev-itpro/developer/al-agent-tools/al-mcp-server
- `al_build`, `al_compile`, `al_publish`, `al_downloadsymbols`, `al_symbolsearch`: see `al-tool-*` pages under the same path
