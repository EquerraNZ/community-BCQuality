---
kind: task-skill
id: bc-mcp-server-data
version: 1
title: Expose BC API pages via the product MCP server
description: Configure Business Central's product MCP server to expose API pages as tools for AI clients (Copilot Studio, Claude, ChatGPT, custom agents). Use when designing which BC entities and operations should be reachable from outside agents.
bc-version: [all]
technologies: [al]
countries: [w1]
application-area: [all]
---

# BC MCP Server (data and business logic)

Business Central runs its own MCP server at `https://mcp.businesscentral.dynamics.com`. This skill is about configuring it: which API pages to expose, what operations to allow, how clients authenticate. Distinct from the AL MCP Server, which is a developer tool.

## When to use

- Designing the surface a Copilot Studio agent, Claude, or ChatGPT should see for a BC tenant
- Deciding which APIs become MCP tools and what CRUD they allow
- Sharing a configuration across customers (export/import JSON)
- Diagnosing "the agent cannot see this entity" or "I can read but cannot write"

## Scope

- BC online (SaaS) only. Not supported on on-prem.
- Endpoint constant: `https://mcp.businesscentral.dynamics.com`. Every customer connects to the same URL with headers selecting the tenant, environment, company, and configuration name.

## Where the configuration lives

Page **8351** in Business Central: `Model Context Protocol (MCP) Server Configurations`. Direct link: https://businesscentral.dynamics.com/?page=8351.

Requires the `MCP - ADMIN` permission set on the user editing configurations.

## Per-configuration switches

| Switch | What it does |
|---|---|
| Active | Configuration is selectable from MCP clients |
| Dynamic Tool Mode | Replaces static tool list with `bc_actions_search`, `bc_actions_describe`, `bc_actions_invoke`. Required if you exceed Copilot Studio's 70-tool cap |
| Discover Additional Objects | Only meaningful when Dynamic Tool Mode is on |
| Unblock Edit Tools | Master switch: when off, all per-API Create/Modify/Delete/Bound Action permissions are ignored (read-only) |

## Per-tool (per-API page) permissions

Each API page added to the configuration has:

- `Allow Read`
- `Allow Create`
- `Allow Modify`
- `Allow Delete`
- `Allow Bound Actions`

Default for a newly-added page is read-only. Enabling write requires `Unblock Edit Tools` to be on at the configuration level AND the specific permission ticked per page.

## Static vs dynamic tool naming

With **Dynamic Tool Mode OFF**, each API page generates up to five static tools:

```
List<EntityName>_PAG<ID>          # if Allow Read
Create<EntityName>_PAG<ID>        # if Allow Create
ListUpdate<EntityName>_PAG<ID>    # if Allow Modify
Delete<EntityName>_PAG<ID>        # if Allow Delete
<BoundActionName>_PAG<ID>         # each allowed bound action
```

With **Dynamic Tool Mode ON**, only three meta-tools are exposed:

```
bc_actions_search        # find available actions by keyword
bc_actions_describe      # get the schema for a specific action
bc_actions_invoke        # call it with parameters
```

This is the way to expose more than 70 tools in Copilot Studio (which caps at 70).

## What pages CANNOT be MCP tools

- API pages of subtype `ListPart` or `CardPart` are not supported.
- Only top-level API pages. Subparts won't be picked up.

If you need to expose a part page's data, add a top-level API page that wraps the same source table.

## Connection string

Get it from page 8351 > **Advanced** > **Connection String**. Shape:

```json
{
  "businesscentral": {
    "url": "https://mcp.businesscentral.dynamics.com",
    "type": "http",
    "headers": {
      "TenantId": "<entra-tenant-guid>",
      "EnvironmentName": "Production",
      "Company": "CRONUS USA, Inc.",
      "ConfigurationName": "MyMCPConfig"
    }
  }
}
```

| Header | Purpose |
|---|---|
| `TenantId` | Entra tenant GUID |
| `EnvironmentName` | BC environment name |
| `Company` | Company within the environment |
| `ConfigurationName` | (Optional) name of the MCP server configuration to use |

## Authentication

OAuth 2.0 Authorization Code with PKCE. Entra ID is the authorization server.

- Microsoft MCP clients (VS Code, Copilot Studio) use a pre-registered application. No setup.
- Non-Microsoft clients (Claude, ChatGPT, custom) must register their own Entra application and configure the MCP client with its client ID.

All operations run as the signed-in user's identity. Audit trails show who did what.

## Export / Import

`Advanced > Export` saves the configuration as JSON. Edit and re-import via `Advanced > Import` to create a new configuration, or share across environments. Useful for promoting configs from dev to test to prod.

## Recommended defaults

When wiring a new customer for AI use:

- Start with one configuration per intended audience (e.g. `SalesTeamConfig`, `WarehouseAgentConfig`). Avoid one mega-configuration.
- Default every API to Read. Open Create/Modify/Delete one entity at a time, only when the agent's workflow requires it.
- Turn on **Dynamic Tool Mode** for any configuration that exceeds 20 APIs. Future-proofs against the 70-tool cap.
- Document each configuration's audience and intended use in a Monday card or the configuration's Description.
- Quarterly review: who has access, what's enabled, is the audit trail clean.

## Common confusions

| Question | Answer |
|---|---|
| Why can the agent read but not create? | `Unblock Edit Tools` is off at the configuration level, OR `Allow Create` is off for that API. Both required. |
| Where is the part page I exposed? | ListPart and CardPart are not supported. Use a top-level API page. |
| Why does Copilot Studio show only 70 tools? | Hard product limit. Turn on Dynamic Tool Mode. |
| Does the agent run as me or as a service account? | As your signed-in user. All operations carry your identity in the audit log. |

## Related skills

- `al-mcp-server` for the developer-tool MCP, distinct from this product MCP
- `copilot-promptdialog` for in-product Copilot UX (different surface)
- `rbac-and-access` for Entra app registration patterns when wiring non-Microsoft clients

## References

- MCP overview: https://learn.microsoft.com/dynamics365/business-central/dev-itpro/ai/mcp-overview
- Configure: https://learn.microsoft.com/dynamics365/business-central/dev-itpro/ai/configure-mcp-server
- MCP spec: https://modelcontextprotocol.io/specification/2025-11-25
