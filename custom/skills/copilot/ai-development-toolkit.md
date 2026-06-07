---
kind: task-skill
id: ai-development-toolkit
version: 1
title: Prototype a custom BC agent with the AI Development Toolkit
description: Prototype and refine custom Business Central agents using the in-product agent design experience and the Tasks AL API. Use when scoping a new agent, defining its instructions, or wiring AL events to trigger agent tasks.
bc-version: [all]
technologies: [al]
countries: [w1]
application-area: [all]
---

# AI Development Toolkit for Business Central

The in-product surface for designing, prototyping, and operating custom BC agents. Pairs with the Agent SDK (`ai-agent-sdk`) for the AL definition, the Evaluation suite (`ai-test-driven-development`) for tests, and the BC MCP Server (`bc-mcp-server-data`) for the data surface the agent operates on.

This is currently a **preview feature** in BC. Preview supplemental terms apply.

## When to use

- Scoping a new agent (Sales Order Agent, Payables Agent, custom)
- Writing or refining the agent's natural-language instructions
- Wiring AL events to create agent tasks (event-driven agents)
- Manually running an agent task from the Agent Tasks page
- Granting or revoking the right to create agents

## Agent design experience

The toolkit gives consultants, product owners, domain experts, and developers a sandbox to prototype agents using natural language instructions. Built for fast iteration:

- Define agent goals and instructions
- Choose the agent profile (which BC role-centre and permissions it has)
- Define access controls (which companies, which API surface)
- Test against real BC data in a sandbox

Same agent runtime as Microsoft's built-in Sales Order and Payables agents.

## Agent instructions

Natural-language goals and guardrails. Prompt-writing best practices apply:

- State the agent's role and scope explicitly
- Provide concrete examples of in-scope and out-of-scope tasks
- List the data sources and tools the agent should use
- Spell out failure handling (when to ask for human intervention vs proceed)

The design experience has an editor with refine-and-test workflow. Iterate before promoting to production.

## Running an agent

Two trigger paths:

1. **Manual**: open the Agent Tasks page in BC, click **Run task**, optionally pass a per-task message that complements the agent's general instructions.
2. **Programmatic**: from AL via the Tasks AL API. Trigger on page actions or business events (email received, sales order posted, etc.). See "Tasks AL API" below.

Tasks queue, can be stopped, and can be restarted. Use insights from past task executions to refine the agent's instructions.

## Tasks AL API (overview)

Live under `BCApps/src/System Application/App/Agent`:

```
codeunit "Custom Agent"
  - GetCustomAgents(var TempAgentInfo: Record "Custom Agent Info" temporary)
    Enumerate custom agents available in this environment.

record "Custom Agent Info" (temporary)
  - "User Security ID" : Guid
  - "User Name"        : Text

codeunit "Agent Session"
  - IsAgentSession(MetadataProvider: Enum "Agent Metadata Provider") : Boolean
    Detect whether the current session is an agent session.

enum "Agent Metadata Provider"
  - ::"Custom Agent"
    Use this value to filter to custom agent sessions.
```

### Detect agent context

```al
local procedure IsCustomAgentRunningThis(): Boolean
var
    AgentSession: Codeunit "Agent Session";
    AgentMetadataProvider: Enum "Agent Metadata Provider";
begin
    exit(AgentSession.IsAgentSession(AgentMetadataProvider::"Custom Agent"));
end;
```

Gate agent-only code paths on this check. For example, suppress an interactive confirm when running as an agent, or capture extra telemetry when the agent fires.

### Enumerate custom agents

```al
local procedure ListAgents()
var
    CustomAgent: Codeunit "Custom Agent";
    TempAgentInfo: Record "Custom Agent Info" temporary;
begin
    CustomAgent.GetCustomAgents(TempAgentInfo);
    if TempAgentInfo.FindSet() then
        repeat
            // TempAgentInfo."User Security ID", TempAgentInfo."User Name"
        until TempAgentInfo.Next() = 0;
end;
```

For agent creation and task orchestration in AL, see `ai-agent-sdk`.

## Permissions and discovery

From **BC 28.1+**, agent discovery is no longer restricted to administrators by default. If you want admin-only agent creation, implement `ShowCanCreateAgent` (in your `IAgentFactory` implementation) and gate on:

```al
AgentSystemPermissions.CurrentUserHasCanManageAllAgentsPermission()
```

Per-user rights are managed via the **Agent Configuration Rights** page.

## Iteration discipline

Treat agent instructions like code:

- Source-control the instructions text (alongside the AL extension that registers the agent)
- Version the instructions when behaviour changes
- Test every change against the Evaluation suite (`ai-test-driven-development`)
- Capture before/after sample task transcripts in the PR

## Related skills

- `ai-agent-sdk` for the AL APIs that define and register agents (`IAgentFactory`, `IAgentMetadata`, `IAgentTaskExecution`)
- `ai-test-driven-development` for agent test data sets, the Evaluation suite, and turn-loop tests
- `copilot-promptdialog` for agent-related UX surfaces in BC
- `bc-mcp-server-data` if the agent's actions should also be reachable from external AI clients

## References

- AI Development Toolkit FAQ: https://learn.microsoft.com/dynamics365/business-central/dev-itpro/ai/ai-development-toolkit-faq
- Integrate with the Tasks AL API: https://learn.microsoft.com/dynamics365/business-central/dev-itpro/ai/ai-development-toolkit-tasks-api
- Run an agent: https://learn.microsoft.com/dynamics365/business-central/dev-itpro/ai/ai-development-toolkit-run-agent
- BCApps Agent source: https://github.com/microsoft/BCApps/tree/main/src/System%20Application/App/Agent
- BCTech Agent and Email Integration sample: https://github.com/microsoft/BCTech
