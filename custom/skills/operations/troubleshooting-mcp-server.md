---
kind: task-skill
id: troubleshooting-mcp-server
version: 1
title: Inspect the AL runtime with the Troubleshooting MCP Server
description: Use the Troubleshooting MCP Server to let Copilot Chat inspect the AL runtime during an active debug session. Use when triaging a runtime error, navigating a complex call stack, or asking why code took a particular path.
bc-version: [all]
technologies: [al]
countries: [w1]
application-area: [all]
---

# Troubleshooting MCP Server for AL

AI-powered debugging surface that lets GitHub Copilot Chat read the AL runtime state during an active debug session. Only available while paused at a breakpoint or runtime error.

## When to use

- A runtime error fired and you want a natural-language explanation that follows the call stack
- The call stack is deep across multiple objects and you want Copilot to summarise it
- You need to know "why did this `if` branch take the false path" without stepping manually
- You want to set a follow-up breakpoint by line number while paused at another one

## When NOT to use

- Interactive step-through debugging (the regular debugger is still better)
- Quick visual inspection of one variable
- Learning unfamiliar code flow (read the code, then debug)

## Prerequisites

- BC **2026 release wave 1 (BC 28) or later**
- Visual Studio Code with the AL Language extension
- GitHub Copilot Chat enabled
- An **active debug session paused at a breakpoint or runtime error**

The server is invisible when no debug session is active. There is no way to replay history.

## The four tools Copilot can call

| Tool | Purpose | Notes |
|---|---|---|
| `Get Stack Frames` | List the call stack | Frame IDs are zero-based: 0 is current, 1 is direct caller, N is N levels up |
| `Get Variables(frameid)` | Variables at a specific frame | Returns `{Name, Value, TypeName, Children}`. `<Uninitialized>` for unset. `<Database Statistics>` for SQL latency/executes/row reads |
| `Get Source Code(frameid)` | Source for a frame | May be empty if the frame is in compiled `.app` files |
| `Add Breakpoint(ApplicationObjectId, ApplicationObjectType, LineNumber)` | Set a breakpoint while paused | E.g. line 42 in codeunit 50100 |

On a runtime error, Copilot will auto-invoke the three diagnostic tools (stack, variables, source) for a comprehensive analysis.

## Forcing its use

Copilot does not always reach for the Troubleshooting MCP. Explicit phrasing helps:

```
Use the Troubleshooting MCP Server to analyze the error at the current breakpoint and suggest a fix
```

```
Add a breakpoint at line 42 in codeunit 50100
```

## What it surfaces that traditional debugging hides

- **Database Statistics** under variable inspection: SQL latency, number of executes, row reads. Great for spotting hidden DB calls in subscribers.
- **Cross-frame analysis**: ask "show me how we got here" and Copilot walks the stack with context.
- **Variable values at any frame**, not just the current one.

## What it does NOT do

- No time-travel. You see what's in scope right now.
- No source for frames whose code is only in compiled `.app` packages (e.g. AppSource dependencies you don't have source for). Fall back to variables inspection.
- No automatic fixes applied. Copilot suggests, you decide.

## Pairing with the AL performance tooling

- For "this is slow but doesn't error" use `performance-profiler` to capture an `.alcpuprofile`.
- For "this errors at runtime and I need to know why" use this MCP server.
- For both: profile first, set breakpoints at the slow frame, then ask the Troubleshooting MCP.

## Common confusions

| Question | Answer |
|---|---|
| Why does Copilot say "no debug session"? | The MCP server is only live during a paused debug session. Start a session, hit a breakpoint, then ask. |
| Why does Get Source Code return empty? | Frame is in compiled `.app` code. Inspect variables instead. |
| Can I use it on BC 26? | No. Requires BC 28 (2026 release wave 1) or later. |

## Related skills

- `al-code-review` for the AL standards Copilot's suggestions should align with
- `performance-profiler` for performance-side triage
- `al-mcp-server` for the headless dev-tool MCP, separate surface

## References

- Troubleshooting MCP Server for AL: https://learn.microsoft.com/dynamics365/business-central/dev-itpro/developer/devenv-debug-mcp-server
