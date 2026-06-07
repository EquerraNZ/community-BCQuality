---
kind: task-skill
id: copilot-capability-implementation
version: 1
title: Implement a BC Copilot capability with System.AI
description: Build the AL side of a Business Central Copilot feature using the System.AI module (Azure OpenAI wrapper). Use when implementing the AOAI call behind a PromptDialog, registering a Copilot capability, choosing billing type, or rotating secrets.
bc-version: [all]
technologies: [al]
countries: [w1]
application-area: [all]
---

# Copilot Capability Implementation (AL → Azure OpenAI)

How to wire the AL side of a Copilot feature: registering the capability, configuring the Azure OpenAI call, constructing prompts, and parsing responses. Pairs with `copilot-promptdialog` for the UX.

## When to use

- Implementing the AL code behind a Copilot feature
- Registering a new Copilot capability so it appears in the Copilot & agent capabilities page
- Choosing between `Microsoft Billed`, `Custom Billed`, and `Not Billed` billing types
- Diagnosing "the capability is not registered" or "billing type validation failed"
- Storing the Azure OpenAI key safely

## Prerequisites

- Azure OpenAI resource: URL (`https://<resourcename>.openai.azure.com/`), Deployment Name, API key
- BC AL runtime with `System.AI` module available
- A registered Copilot Capability enum value (see "Register the capability" below)

The `System.AI` module wraps Azure OpenAI. Supports text completion, chat completion, embeddings. No DALL-E (images), no Whisper (speech).

## Register the capability

Every Copilot feature must extend the `Copilot Capability` enum AND register at install time. Without registration, calls fail with "capability not registered".

### Extend the enum

```al
enumextension 50300 "My Copilot Caps" extends "Copilot Capability"
{
    value(50301; "Draft a Job") { Caption = 'Draft a Job'; }
}
```

### Register on install

```al
codeunit 50302 "My Copilot Install"
{
    Subtype = Install;
    InherentEntitlements = X;
    InherentPermissions = X;

    trigger OnInstallAppPerDatabase()
    var
        CopilotCapability: Codeunit "Copilot Capability";
        EnvironmentInfo: Codeunit "Environment Information";
        LearnMoreUrlTxt: Label 'https://example.com/copilot/draft-a-job', Locked = true;
    begin
        if not EnvironmentInfo.IsSaaSInfrastructure() then
            exit;
        if CopilotCapability.IsCapabilityRegistered(Enum::"Copilot Capability"::"Draft a Job") then
            exit;
        CopilotCapability.RegisterCapability(
            Enum::"Copilot Capability"::"Draft a Job",
            Enum::"Copilot Availability"::"Generally Available",
            Enum::"Copilot Billing Type"::"Microsoft Billed",
            LearnMoreUrlTxt);
    end;
}
```

| Parameter | Values |
|---|---|
| Availability | `Preview`, `Generally Available` |
| Billing Type | `Custom Billed` (partner billed), `Microsoft Billed` (Microsoft billed, required if using BC AI resources), `Not Billed` |

**Always guard with `IsSaaSInfrastructure()` and `IsCapabilityRegistered()`** before calling `RegisterCapability`. Duplicate registration throws.

## Billing type validation

Runtime validates the billing type against actual Azure OpenAI usage. Disallowed combinations:

| Partner billing type | BC AI resources | Own AOAI resource |
|---|---|---|
| `Microsoft Billed` | Production OK | Sandbox only |
| `Custom Billed` | **Never allowed** | OK |
| `Not Billed` | OK (no consumption) | OK (no consumption) |

Pick `Microsoft Billed` if your customers use BC AI credits. Pick `Custom Billed` if you bring your own AOAI deployment. Pick `Not Billed` only for trial/demo capabilities that should not show in customer billing.

## Store the API key

```al
table 50301 "My Copilot Setup"
{
    DataClassification = SystemMetadata;

    fields
    {
        field(1; "Primary Key"; Code[10]) { }
        field(2; Endpoint;     Text[250]) { }
        field(3; Deployment;   Text[250]) { }
        // Note: ApiKey is NOT a regular field.
    }
}
```

The API key field **must be `SecretText` type** in AL. This excludes it from the debugger.

Persist via `IsolatedStorage`:

```al
procedure SetApiKey(NewKey: SecretText)
begin
    IsolatedStorage.Set('AOAI_KEY', NewKey, DataScope::Module);
end;

procedure GetApiKey() Result: SecretText
begin
    if not IsolatedStorage.Get('AOAI_KEY', DataScope::Module, Result) then
        Error('AOAI key not configured.');
end;
```

For marketplace apps, **AppSource Key Vault** is the alternative. Customers do not bring their own keys; Microsoft injects them at runtime.

## Make the AOAI call

```al
codeunit 50303 "Generate Job Proposal"
{
    procedure Run(JobDescription: Text) Result: Text
    var
        AzureOpenAI: Codeunit "Azure OpenAI";
        AOAIChatMessages: Codeunit "AOAI Chat Messages";
        AOAIChatCompletionParams: Codeunit "AOAI Chat Completion Params";
        AOAIOperationResponse: Codeunit "AOAI Operation Response";
        Endpoint: Text;
        Deployment: Text;
        ApiKey: SecretText;
        Metaprompt: Text;
        UserPrompt: Text;
    begin
        // 1. Identify the capability so credit tracking and gating work
        AzureOpenAI.SetCopilotCapability(Enum::"Copilot Capability"::"Draft a Job");

        // 2. Load credentials
        LoadConnectionDetails(Endpoint, Deployment, ApiKey);

        // 3. Configure auth
        AzureOpenAI.SetAuthorization(
            Enum::"AOAI Model Type"::"Chat Completions",
            Endpoint, Deployment, ApiKey);

        // 4. Set generation parameters
        AOAIChatCompletionParams.SetMaxTokens(2500);
        AOAIChatCompletionParams.SetTemperature(0.3);

        // 5. Build the conversation
        Metaprompt := GetMetaprompt();              // role, scope, format rules
        AOAIChatMessages.SetPrimarySystemMessage(Metaprompt);
        AOAIChatMessages.AddUserMessage(JobDescription);

        // 6. Generate
        AzureOpenAI.GenerateChatCompletion(
            AOAIChatMessages, AOAIChatCompletionParams, AOAIOperationResponse);

        if not AOAIOperationResponse.IsSuccess() then
            Error('Copilot could not generate a result.');

        Result := AOAIChatMessages.GetLastMessage();
    end;
}
```

## Primary system message vs regular system message

```al
AOAIChatMessages.SetPrimarySystemMessage(Metaprompt);     // persists across history
AOAIChatMessages.AddSystemMessage('Extra guidance');      // does NOT persist
```

The **PrimarySystemMessage persists across the chat history** even when older turns are evicted. Use it for the metaprompt (role, format, guardrails). Use regular system messages for transient hints.

## Token budgeting

```al
local procedure FitsInContextWindow(MessagesTotalText: Text; MaxResponse: Integer): Boolean
var
    AzureOpenAI: Codeunit "Azure OpenAI";
    EstTokens: Integer;
    ContextWindow: Integer;
begin
    ContextWindow := 4096;            // GPT 3.5 Turbo; check your model
    EstTokens := AzureOpenAI.ApproximateTokenCount(MessagesTotalText);
    exit(EstTokens + MaxResponse <= ContextWindow);
end;
```

Reserve enough for the response. The BC sample reserves 2500 tokens for output on a 4096-token model.

## AOAI model types

```
Enum "AOAI Model Type"::
  - "Embeddings"
  - "Text Completions"
  - "Chat Completions"   // recommended for most Copilot features
```

## Chat roles

```
Enum "AOAI Chat Roles"::
  - User
  - System
  - Assistant
```

## Common gotchas

| Symptom | Cause | Fix |
|---|---|---|
| "Capability not registered" | `RegisterCapability` never called or wrong enum value | Wrap registration in `OnInstallAppPerDatabase`, guard with `IsCapabilityRegistered` |
| Billing validation fails | `Custom Billed` + BC AI resources, or `Microsoft Billed` + own resource in production | Re-check the matrix above |
| API key visible in debugger | Used `Text` instead of `SecretText` | Change field type and reload |
| `SetPrimarySystemMessage` content disappears mid-conversation | Used `AddSystemMessage` instead | Use `SetPrimarySystemMessage` for content that must persist |
| Response truncated | `SetMaxTokens` too low or token budget exceeded | Lower input size or raise `SetMaxTokens`, check context window |

## Note on the "Chat with Copilot" preview

The in-product "Chat with Copilot" feature is **not extensible**. It's a separate UI Microsoft owns. AL chat completion is the API; the chat experience surfaced by the capability is the PromptDialog you build.

## Related skills

- `copilot-promptdialog` for the UI side
- `ai-test-driven-development` for testing the generated output
- `ai-agent-sdk` if the feature is an agent (not a one-shot capability)

## References

- Build the Copilot capability in AL: https://learn.microsoft.com/dynamics365/business-central/dev-itpro/developer/ai-build-capability-in-al
- AzureOpenAI codeunit reference: see System Application repo at https://github.com/microsoft/BCApps
- Azure OpenAI System message framework: https://learn.microsoft.com/azure/ai-services/openai/concepts/system-message
- Azure OpenAI REST API: https://learn.microsoft.com/azure/ai-services/openai/reference
- BC sample (Suggest Job): https://github.com/microsoft/BCTech/blob/master/samples/AzureOpenAI/Advanced_SuggestJob
