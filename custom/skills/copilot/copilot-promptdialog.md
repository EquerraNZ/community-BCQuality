---
kind: task-skill
id: copilot-promptdialog
version: 1
title: Design BC Copilot UX with PromptDialog
description: Design Business Central Copilot UX using the PromptDialog page type. Use when building a new Copilot feature's UI, reviewing a PromptDialog implementation, or diagnosing "the Generate button does not appear" / "trailing whitespace breaks Copilot" issues.
bc-version: [all]
technologies: [al]
countries: [w1]
application-area: [all]
---

# Copilot PromptDialog Page Type

The only sanctioned UI surface for Copilot features in Business Central. PromptDialog gives users a structured input/output flow with the signature Copilot look, system actions, prompt guides, and built-in safety controls.

## When to use

- Building a new Copilot feature with an interactive Prompt → Generate → Content flow
- Reviewing a `PageType = PromptDialog` implementation for compliance
- Diagnosing why the Generate button does not appear or why Copilot stops responding mid-action
- Wiring a prompt guide and prompt options

## Prerequisites

- BC AL runtime **12.1 or later**
- The feature must already have a registered Copilot Capability (see `copilot-capability-implementation`)

## Snippet to scaffold

In VS Code with the AL Language extension:

```
tpage → "Page of type Prompt Dialog"
```

## Required page properties

```al
page 54320 "Copilot Job Proposal"
{
    PageType = PromptDialog;
    Extensible = false;           // mandatory
    IsPreview = true;             // optional, adds an in-UI preview note
    Image = Sparkle;              // or SparkleFilled for prominent options
    Caption = 'Suggest Job';

    layout { ... }
    actions { ... }
}
```

| Property | Why it matters |
|---|---|
| `PageType = PromptDialog` | The whole UX framework is gated on this type. No other page type renders the signature Copilot frame. |
| `Extensible = false` | **Mandatory.** Customers cannot extend Copilot pages. This protects the AI experience from drift. |
| `Image = Sparkle` (or `SparkleFilled`) | Standard Copilot icon. Use `SparkleFilled` for an action that should stand out. |
| `IsPreview = true` | Adds a "preview" note in the UI. Set during the feature's preview lifecycle. |
| `PromptMode` | Switches the page between `Prompt`, `Generate`, and `Content` mode. Default starts in `Prompt`. Set `CurrPage.PromptMode` before page opens to override. |
| `DataCaptionExpression` | Optional title customisation per generated result. |

## Layout areas (only three supported)

```al
layout
{
    area(Prompt)
    {
        // Input the user provides. Free-text fields, dropdowns, etc.
        // NO repeater controls.
    }
    area(Content)
    {
        // Output the AI produced. NO repeater controls.
    }
    area(PromptOptions)
    {
        // Option-type fields ONLY. Renders as buttons next to system actions.
        field(tone; Tone) { }
        field(format; Format) { }
    }
}
```

- `Prompt`, `Content`, and `PromptOptions` are the only allowed area types in a PromptDialog page.
- **No repeater controls** in `Prompt` or `Content`. Use a list or array represented as a single field.
- `PromptOptions` accepts only fields of the **option data type**.

## Action areas (only two supported)

```al
actions
{
    area(SystemActions)
    {
        systemaction(Generate)
        {
            Caption = 'Generate';
            trigger OnAction()
            begin
                RunGeneration();   // your AOAI call lives here
            end;
        }
        systemaction(Regenerate) { Caption = 'Try again'; trigger OnAction() begin RunGeneration(); end; }
        systemaction(Attach)     { Caption = 'Attach file'; }
        systemaction(OK)         { Caption = 'Keep it'; }
        systemaction(Cancel)     { Caption = 'Discard'; }
    }

    area(PromptGuide)
    {
        // Predefined prompt texts. Only rendered when PromptMode = Prompt.
        action(SuggestForRetailer)
        {
            Caption = 'Suggest for a retailer';
            ToolTip = 'Use this when the customer is a retail business.';
            trigger OnAction()
            begin
                InputProjectDescription := 'Draft a marketing email for a retailer that ...';
            end;
        }
    }
}
```

- Only `SystemActions` and `PromptGuide` areas are valid action areas.
- The five system actions: `Generate`, `Regenerate`, `Attach`, `OK`, `Cancel`. No custom system actions.
- `OK` is the "Keep it" action. `Cancel` is "Discard".
- `PromptGuide` actions typically set the input variable to a templated text in `OnAction`. They render only when `PromptMode = Prompt`.

## OnQueryClosePage pattern

Persist the result when the user clicks OK:

```al
trigger OnQueryClosePage(CloseAction: Action): Boolean
begin
    if CloseAction = Action::OK then
        SaveGeneratedContent();
    exit(true);
end;
```

## Error handling with retry

The official sample uses a retry loop wrapping the AOAI call:

```al
local procedure RunGeneration()
var
    GenJobProposal: Codeunit "Generate Job Proposal";
    Attempts: Integer;
    GenerationFailed: Label 'Copilot could not generate a result. Try again or rephrase your prompt.';
begin
    for Attempts := 0 to 3 do
        if GenJobProposal.Run() then begin
            // success, propagate the generated content into the page
            exit;
        end;
    Error(GenerationFailed);
end;
```

The pattern: zero-indexed up to N attempts, each `Codeunit.Run()` swallows errors so you can retry, terminal Error with a friendly label.

## Anti-patterns to flag in code review

- **Trailing whitespace in action names.** Breaks Copilot silently. Caption can have trailing space, but Name cannot.
- **`Extensible = true`** on a PromptDialog. Mandatory `false`.
- **Repeater control inside `area(Prompt)` or `area(Content)`.** Not supported. Use list-of-text or a JSON shape rendered as a multiline field.
- **Custom system actions.** Only the five sanctioned names work. Don't try to register your own.
- **Non-option field in `PromptOptions`.** Only option fields. Use enums.
- **Skipping `IsPreview`** during preview releases. Adds a user-facing note that this is preview.
- **Missing prompt guide on a complex feature.** Users don't know how to phrase prompts. Provide at least 3 examples.

## Nudging users toward Copilot via floating action bar

Use a prompt action on a normal page to promote a Copilot feature. The floating action bar surfaces relevant Copilot features in context. See `devenv-page-prompting-floating-actionbar` in the AL docs.

## Related skills

- `copilot-capability-implementation` for the underlying AL Copilot capability that the page calls
- `ai-test-driven-development` for testing PromptDialog features end-to-end
- `al-code-review` for the wider AL standards the page must follow

## References

- The PromptDialog page type: https://learn.microsoft.com/dynamics365/business-central/dev-itpro/developer/devenv-page-type-promptdialog
- Build Copilot user experience: https://learn.microsoft.com/dynamics365/business-central/dev-itpro/developer/ai-build-experience
- BCTech CopilotJobProposal sample: https://github.com/microsoft/BCTech/blob/master/samples/AzureOpenAI/Advanced_SuggestJob/DescribeJob/CopilotJobProposal.Page.al
- Best practices for AL code (action names): https://learn.microsoft.com/dynamics365/business-central/dev-itpro/compliance/apptest-bestpracticesforalcode
