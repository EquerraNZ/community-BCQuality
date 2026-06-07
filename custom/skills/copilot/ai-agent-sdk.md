---
kind: task-skill
id: ai-agent-sdk
version: 1
title: Define a custom BC agent with the AI Agent SDK
description: Define and register a custom Business Central agent in AL using the IAgentFactory, IAgentMetadata, and IAgentTaskExecution interfaces. Use when authoring a new agent extension, wiring user intervention suggestions, or registering the paired Copilot capability.
bc-version: [all]
technologies: [al]
countries: [w1]
application-area: [all]
---

# AI Agent SDK (define and register an agent in AL)

How to define a custom Business Central agent programmatically in AL. Pairs with `ai-development-toolkit` (which covers the in-product design surface and Tasks AL API) and `ai-test-driven-development` (for evaluating the agent).

Currently a **preview feature**. Preview supplemental terms apply.

## When to use

- Authoring a new BC agent extension
- Adding a setup page, custom annotations, or intervention suggestions to an existing agent
- Registering the paired Copilot Capability so the agent appears in the Copilot & agent capabilities page
- Replacing a built-in agent's behaviour with a customer-tuned variant

## The triple-interface pattern

Every agent type plugs into three interfaces. Register them via an `enumextension` of `Agent Metadata Provider`:

```al
enumextension 50100 "My Agent Provider" extends "Agent Metadata Provider"
{
    value(50101; "My Agent")
    {
        Implementation =
            IAgentFactory = MyAgentFactory,
            IAgentMetadata = MyAgentMetadata,
            IAgentTaskExecution = MyAgentTaskExecution;
    }
}
```

| Interface | Concern | Typical methods |
|---|---|---|
| `IAgentFactory` | Creation flow and UI gates | `GetFirstTimeSetupPageId`, `ShowCanCreateAgent`, `GetDefaultProfile`, `GetDefaultAccessControls` |
| `IAgentMetadata` | Runtime metadata that BC needs about the agent | `GetSetupPageId`, `GetAgentTaskMessagePageId`, `GetAgentAnnotations`, `GetSummaryPageId` |
| `IAgentTaskExecution` | Per-turn input/output analysis and intervention | `AnalyzeAgentTaskMessage`, `GetAgentTaskUserInterventionSuggestions` |

## IAgentFactory

- `GetFirstTimeSetupPageId`: the page BC opens when a user creates a new instance of this agent type. The source table on that page **must** contain a `User Security ID : Guid` field. BC injects the new agent's user id into that field.
- `ShowCanCreateAgent`: returns whether the current user can create this agent type. From BC 28.1+, agent discovery is no longer admin-only by default. Gate on `AgentSystemPermissions.CurrentUserHasCanManageAllAgentsPermission()` to keep admin-only.
- `GetDefaultProfile(var TempAllProfile: Record "All Profile" temporary)`: which role-centre profile the agent uses by default.
- `GetDefaultAccessControls(var TempAccessControlBuffer: Record "Access Control Buffer" temporary)`: default permission sets to assign on creation.

## IAgentMetadata

- `GetSetupPageId`: the page BC opens when a user views or edits an existing agent.
- `GetAgentTaskMessagePageId`: defaults to `Page::"Agent Task Message Card"`. Override only if your agent needs a custom task message UI.
- `GetAgentAnnotations(var Annotations: Record "Agent Annotation")`: returns the static annotations to attach to every task message of this agent type. Annotations carry severity, code, message, details.
- `GetSummaryPageId`: a numeric-only KPI summary page shown on the agent's dashboard tile. Keep it focused: this is the at-a-glance health view.

## IAgentTaskExecution

This is where the runtime hooks live.

- `AnalyzeAgentTaskMessage(var AgentTaskMessage: Record "Agent Task Message")`:
  - Called for both `Type::Input` AND `Type::Output` messages.
  - You can mutate the output (e.g. append a signature, normalise dates).
  - Attach annotations to flag issues:
    - `Severity::Error` halts the task with the annotation message
    - `Severity::Warning` requests user intervention (a human reviews before continuing)
  - Use `codeunit "Agent Message".UpdateText(AgentTaskMessage, NewText)` to mutate text safely.

- `GetAgentTaskUserInterventionSuggestions(var AgentTaskUserIntSuggestion: Record "Agent Task User Int Suggestion")`:
  - Called when a Warning annotation triggers intervention.
  - Provide one or more suggestions: each has `Summary`, `Description` (Locked), `Instructions` (what the human should review).
  - Example: "Customer requested an unusual discount" → suggestions: "Approve and continue", "Reject", "Request supporting documentation".

### Example: append a signature to every output

```al
codeunit 50102 MyAgentTaskExecution implements IAgentTaskExecution
{
    procedure AnalyzeAgentTaskMessage(var AgentTaskMessage: Record "Agent Task Message")
    var
        AgentMessage: Codeunit "Agent Message";
        OldText: Text;
        SignatureTxt: Label '\n\nSent by your AI agent.', Locked = true;
    begin
        if AgentTaskMessage.Type <> AgentTaskMessage.Type::Output then
            exit;
        OldText := AgentMessage.GetText(AgentTaskMessage);
        if not OldText.EndsWith(SignatureTxt) then
            AgentMessage.UpdateText(AgentTaskMessage, OldText + SignatureTxt);
    end;

    procedure GetAgentTaskUserInterventionSuggestions(var AgentTaskUserIntSuggestion: Record "Agent Task User Int Suggestion")
    begin
        // ...
    end;
}
```

## Register the paired Copilot Capability

Every custom agent must have a matching `Copilot Capability` enum value AND a runtime registration. Without registration the agent will not appear in the Copilot & agent capabilities page and will refuse to run.

### Extend the capability enum

```al
enumextension 50103 "My Copilot Caps" extends "Copilot Capability"
{
    value(50110; "My Agent Capability") { Caption = 'My Agent'; }
}
```

### Register at install time

```al
codeunit 50104 "My Agent Install"
{
    Subtype = Install;
    InherentEntitlements = X;
    InherentPermissions = X;

    trigger OnInstallAppPerDatabase()
    var
        CopilotCapability: Codeunit "Copilot Capability";
        LearnMoreUrlTxt: Label 'https://example.com/agents/my-agent', Locked = true;
    begin
        if not CopilotCapability.IsCapabilityRegistered(Enum::"Copilot Capability"::"My Agent Capability") then
            CopilotCapability.RegisterCapability(
                Enum::"Copilot Capability"::"My Agent Capability",
                Enum::"Copilot Availability"::Preview,
                Enum::"Copilot Billing Type"::"Microsoft Billed",
                LearnMoreUrlTxt);
    end;
}
```

Availability values: `Preview`, `Generally Available`. Billing values: `Custom Billed`, `Microsoft Billed`, `Not Billed`.

## Permission rules (BC 28.1+)

- Default behaviour: any user can discover and (subject to setup gates) create agent instances.
- To enforce admin-only: implement `ShowCanCreateAgent` to return `AgentSystemPermissions.CurrentUserHasCanManageAllAgentsPermission()`.
- Per-user agent rights are managed through the **Agent Configuration Rights** page.

## Anti-patterns

- Skipping the `IsCapabilityRegistered` check before `RegisterCapability`: duplicate registration throws.
- Returning `false` from `ShowCanCreateAgent` without an alternative path: the agent type becomes uncreatable until you fix the gate.
- Mutating the input message in `AnalyzeAgentTaskMessage`: only mutate `Type::Output`. Input is the user's record.
- Raising annotations with `Severity::Error` on every minor issue: prefer Warning + intervention suggestions. Errors halt the task.
- Forgetting that the setup page's source table needs a `User Security ID : Guid` field.

## Related skills

- `ai-development-toolkit` for the design surface and Tasks AL API
- `ai-test-driven-development` for evaluation suites and agent tests
- `copilot-promptdialog` if your agent surfaces a Copilot prompt experience
- `copilot-capability-implementation` for capabilities that wrap a direct Azure OpenAI call (no agent runtime)

## References

- Define and register an agent: https://learn.microsoft.com/dynamics365/business-central/dev-itpro/ai/ai-agent-sdk-define-register
- BCApps Agent source: https://github.com/microsoft/BCApps/tree/main/src/System%20Application/App/Agent
- BCTech Sales Validation Agent sample: https://github.com/microsoft/BCTech/tree/master/samples/BCAgents/SalesValidationAgent
