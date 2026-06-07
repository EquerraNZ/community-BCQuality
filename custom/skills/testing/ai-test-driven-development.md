---
kind: task-skill
id: ai-test-driven-development
version: 1
title: Test-driven development for BC Copilot and agents
description: Test-driven development for Business Central Copilot features and custom agents. Covers the Evaluation suite, JSONL and YAML datasets, the AITest codeunit pattern, agent turn loops, intervention validation, and Copilot credit tracking.
bc-version: [all]
technologies: [al]
countries: [w1]
application-area: [all]
---

# AI Test-Driven Development for AL

How to test Copilot features and agents in Business Central. Uses the Evaluation tool (formerly "AI Test Toolkit"), which is data-driven: datasets describe inputs and expected outputs, the test codeunit drives the loop.

## When to use

- Building a new Copilot feature (PromptDialog) and want a regression suite
- Building a custom agent and want repeatable accuracy tests
- Reviewing a PR that changes prompts, metaprompts, or agent instructions
- Comparing two model versions (A/B) on the same dataset
- Measuring Copilot credit consumption per dataset entry

## Two test flows

| Flow | Use for | Format |
|---|---|---|
| **AI test** | Prompt-based Copilot features (PromptDialog + AOAI call) | JSONL or YAML |
| **Agent test** | Multi-turn custom agents with intervention | YAML only |

Both use the same Evaluation suite UI and runner. Differences come down to dataset shape and the codeunit's `TestType`.

## Test codeunit attributes

For an agent test, use:

```al
codeunit 50200 "My Agent Accuracy Test"
{
    Subtype = Test;
    TestType = AITest;
    TestPermissions = Disabled;
    RequiredTestIsolation = Disabled;
}
```

`RequiredTestIsolation = Disabled` is essential. Agent tasks run in a different session and span transactions, so isolation cannot be enforced.

For a prompt-only AI test, the codeunit still uses `TestType = AITest` but does not need the disabled isolation, because the test runs inline.

## Required codeunits

```
codeunit "AIT Test Context"           # read dataset values
codeunit "Library - Agent"            # turn dispatcher (agent tests only)
codeunit "Library Assert"             # standard assertions
codeunit "Test Input Json"            # parse per-turn setup data
codeunit "Agent Task Builder"         # build agent tasks programmatically
codeunit "Agent Task Message Builder" # build messages programmatically
```

## The agent test turn loop

The recommended pattern: dataset describes the input and expected outcome, the test runs a `repeat ... until` loop delegating to `Library - Agent`.

```al
[Test]
procedure RunAgentTurns()
var
    AITTestContext: Codeunit "AIT Test Context";
    LibraryAgent: Codeunit "Library - Agent";
    AgentTask: Record "Agent Task";
    TurnSuccessful: Boolean;
    ContinueWithNextTurn: Boolean;
    ErrorReason: Text;
begin
    Initialize();   // resolve agent, clean up old tasks, activate

    repeat
        ApplyTurnSetup();   // optional: per-turn setup from dataset

        TurnSuccessful := LibraryAgent.RunTurnAndWait(AgentUserSecurityId, AgentTask);
        if TurnSuccessful then
            TurnSuccessful := ValidateTurnCompletedSuccessfully(ErrorReason);

        ContinueWithNextTurn := LibraryAgent.FinalizeTurn(AgentTask, TurnSuccessful, ErrorReason);
    until not ContinueWithNextTurn;
end;
```

Key calls:

- `LibraryAgent.RunTurnAndWait(AgentUserSecurityId, var AgentTask): Boolean` reads the current turn's `query:` from the dataset, dispatches to the agent, waits for completion.
- `LibraryAgent.FinalizeTurn(var AgentTask, TurnSuccessful, ErrorReason): Boolean` writes turn output, validates intervention contract, advances to next turn. Return value drives loop continuation.

Validators should return `false` with a populated `ErrorReason` rather than calling `Error()`. That lets `FinalizeTurn` log the failure on the turn and optionally continue.

## Initialize and agent resolution

```al
local procedure Initialize()
var
    LibraryAgent: Codeunit "Library - Agent";
begin
    if Initialized then exit;

    AgentUserSecurityId := LibraryAgent.GetAgentUnderTest();
    if IsNullGuid(AgentUserSecurityId) then
        AgentUserSecurityId := GetOrCreateAgent();

    LibraryAgent.StopTasks(AgentUserSecurityId);
    LibraryAgent.EnsureAgentIsActive(AgentUserSecurityId);
    Initialized := true;
end;
```

`GetAgentUnderTest()` is optional. Use it when the evaluation suite should A/B test two agent versions against the same dataset. Skip it for tests that always run against one agent.

## Dataset shapes

### AI test dataset (JSONL or YAML)

```yaml
name: MARKETING-TEXT
description: Marketing text generation accuracy tests.
language: en-US
tests:
  - test_setup:
      item_no: "C-10000"
      description: "Contoso Coffee Machine"
      uom: "PCS"
    expected_data:
      tagline_max_length: 20
```

`test_setup` and `expected_data` are conventional keys (the framework only enforces a few; you read the rest in your validator).

### Agent test dataset (YAML)

Agent datasets live under conventional folders (the names are convention, not enforced):

```
.resources/
    suite_setup/<NAME>.yaml      # suite-level setup, declared once
    datasets/<NAME>.yaml         # per-suite test cases
    configuration/<NAME>.xml     # AI Eval Suite XML
```

Setup file:

```yaml
name: MY-AGENT
suite_setup:
  setup_actions:
    - action_type: SeedCustomers
      action_data:
        count: 5
```

Dataset file (always uses `turns:` even for single-turn):

```yaml
name: MY-DATASET
suite_setup: MY-AGENT
language: en-US
continue_on_failure: false
tests:
  - turns:
      - query:
          from: Jane Doe
          title: "Release orders"
          message: "Release all open sales orders for the next week"
          attachments:
            - file: invoices/inv-001.pdf
        expected_data:
          orders_released: 2
```

### Intervention validation (framework-recognised)

`expected_data.intervention_request` is the only sub-key the framework reads automatically:

```yaml
expected_data:
  intervention_request:
    type: Assistance      # enum "Agent User Int Request Type" English name
    suggestions: [PROVIDE_DATE]
```

`FinalizeTurn` enforces both directions:

- If the turn declares `intervention_request`, the agent **must** pause with matching type and suggestions. Failure to pause = turn fails.
- If the turn does **not** declare it, the agent **must not** pause. Unexpected pause = turn fails.

### Continuing past an intervention

A later turn can resume from the previous intervention via `query.intervention`:

```yaml
- query:
    intervention:
      suggestion: PROVIDE_DATE
```

Use either `suggestion` (resume one of the offered suggestions) or `instruction` (free-text override), not both.

### Date placeholders

`$DateFormula-<formula>$` resolves against `WorkDate` so tests do not drift.

```yaml
Shipment Date: "$DateFormula-<CW+1M>$"
Posting Date: "$DateFormula-<CD>$"
```

Variants: `$DateTimeFormula-<formula>$`, `$DateTimeFormula-<formula>-HH:MM:SS$`, plus a milliseconds variant.

**Always quote these strings** in YAML. The `< >` characters conflict with YAML flow syntax otherwise.

## Suite XML

Drives which tests in which languages on what cadence.

```xml
<AITSuite Code="MY-AGENT"
          Description="My agent accuracy suite"
          Dataset="MY-DATASET.YAML"
          TestRunnerId="130451"
          Capability="My Agent Capability"
          Frequency="Daily"
          TestType="Agent">
  <Language Tag="en-US" Frequency="Daily"/>
  <Language Tag="da-DK" Frequency="Weekly"/>
  <Line CodeunitID="50200" Dataset="MY-DATASET.YAML"/>
</AITSuite>
```

- `TestRunnerId="130451"` is `Test Runner - Isol. Disabled`, required for agent tests.
- `TestType="Agent"` opts into the agent runner. Use `"AITest"` for prompt-only tests.
- `<Language>` children enable multilingual evaluation.

## Suite setup discipline

`AITTestContext.IsSuiteSetupDone()` is **sticky** across runs. Once `SetEvalSuiteSetupCompleted()` is called, the suite skips setup on subsequent runs.

To re-run setup (e.g. after editing the setup YAML), use the **Reset Suite Setup** action on the AI Eval Suite page. Otherwise the new setup is ignored.

## Loading datasets at install time

The convention is to ship datasets as test app `.resources/` files and load them in an Install codeunit:

```al
codeunit 50201 "My Agent Test Install"
{
    Subtype = Install;

    trigger OnInstallAppPerDatabase()
    var
        AITALTestSuiteMgt: Codeunit "AIT AL Test Suite Mgt";
        ResInStream: InStream;
        ResourcePath: Text;
    begin
        foreach ResourcePath in NavApp.ListResources('*.yaml') do begin
            NavApp.GetResource(ResourcePath, ResInStream);
            AITALTestSuiteMgt.ImportTestInputs(ResourcePath, ResInStream);
        end;
        // and the suite XML
    end;
}
```

## Copilot credit tracking

Evaluation runs **consume Copilot credits**. The runner tracks usage at three levels:

- Per suite run
- Per test line
- Per dataset entry

Use environments with prepaid Copilot credits, especially for automated runs.

Credit limits are enforced at **two levels**:

- Environment (all companies combined)
- Company (per company)

Either limit blocks new tasks. Running tasks complete to avoid wasted credits.

For agent tests, the displayed token usage shows **AI evaluator tokens only**, not the agent's runtime tokens. Don't confuse the two when budgeting.

## Access to results via API

Page `149038` `AIT Log Entry API` exposes results programmatically. Useful for CI dashboards.

## Sensitive datasets

Toggle the `Sensitive` flag on a dataset to hide test input/output in views by default. Useful when the dataset contains PII or proprietary prompts.

## Permissions

Users running Evaluation need the `AI TEST TOOLKIT` permission set. Credit limit edits require the `agent admin` role.

## BC-Bench (April 2026 GA)

A SWE-Bench-style benchmark for AL bug fix and test creation tasks. Use it for comparing Copilot agent performance over time. Out of scope for project-specific tests but useful as the trust signal when evaluating agent providers.

## Related skills

- `ai-agent-sdk` for the AL APIs that define the agents under test
- `ai-development-toolkit` for the design and Tasks API context
- `copilot-promptdialog` for the UI under test
- `copilot-capability-implementation` for the AL Copilot capability the tests evaluate

## References

- Evaluation: https://learn.microsoft.com/dynamics365/business-central/dev-itpro/developer/ai-test-copilot-testtool
- Datasets: https://learn.microsoft.com/dynamics365/business-central/dev-itpro/developer/ai-test-copilot-datasets
- Write agent tests: https://learn.microsoft.com/dynamics365/business-central/dev-itpro/developer/ai-test-copilot-agent-tests
- BC-Bench (release plan): https://learn.microsoft.com/dynamics365/release-plan/2026wave1/smb/dynamics365-business-central/evaluate-al-coding-agents-bc-bench
- BCApps AI Test Toolkit README: https://github.com/microsoft/BCApps/blob/main/src/Tools/AI%20Test%20Toolkit/README.md
- BCTech SalesValidationAgent sample: https://github.com/microsoft/BCTech/tree/master/samples/BCAgents/SalesValidationAgent
