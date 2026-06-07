---
kind: task-skill
id: performance-profiler
version: 1
title: Capture and interpret BC performance profiles
description: Capture and interpret Business Central performance profiles using the built-in Analyze Performance Profiler. Use when triaging slow pages, slow posting, intermittent slowness, or when asked to provide performance evidence to a partner.
bc-version: [all]
technologies: [al]
countries: [w1]
application-area: [all]
---

# BC Performance Profiler

How to capture and read a Business Central `.alcpuprofile` performance snapshot. This is the evidence base for any serious AL performance triage.

## When to use

- A user reports a slow page, slow report, or slow posting
- Intermittent slowness ("it was fast yesterday")
- Need to prove or disprove that a specific extension is causing slowness
- Microsoft Support or another partner has asked for a profile

## Capture (customer-side, ~2 minutes)

1. In Business Central, click the **Help & Support** icon (the `?`).
2. Select **Troubleshooting** > **Analyze Performance**.
3. Click **Start** on the Performance Profiler page.
4. Reproduce the slow action exactly (open the slow page, post the slow document, run the slow report).
5. Click **Stop**.
6. The profiler shows results. Click **Download Profile** to save the `.alcpuprofile` file.
7. Send the file to the technical consultant or attach to the support ticket.

Notes:
- The profiler captures Microsoft first-party apps **plus** all installed third-party extensions. Nothing is hidden from the partner.
- The `.alcpuprofile` file is plain JSON. It is safe to attach to a ticket as-is.
- A profile of a 5 to 30 second action is the sweet spot. Longer captures get noisy.

## Interpret (consultant-side)

Open the `.alcpuprofile` file in **VS Code with the AL Language extension installed**, or re-upload it to BC via the same Analyze Performance page for the graphical view.

You will see these views:

| View | What it answers |
|---|---|
| **Active Apps** chart | Which apps consumed CPU time during the capture |
| **Time Spent** chart | Timeline view, where the spikes are |
| **Aggregate Results** | Top expensive functions across the whole capture |
| **Time Spent** pane | Per-frame breakdown |
| **Time Spent by Application Object** | Which BC object types (table, codeunit, page) ate the time |
| **Call Tree** | Drill into a specific code path |

### How to read it

1. **Start with Active Apps**. If one app dominates, that is your suspect.
2. **Cross-check Time Spent**. Is the cost continuous (likely AL inefficiency) or spiky (likely SQL, lock, or external service)?
3. **Aggregate Results** gives the top functions. Anything in the top 10 that is yours, fix first.
4. **Call Tree** shows the chain. A slow `FindSet` deep in a subscriber chain shows up here, not in the aggregate top.
5. **By Application Object** tells you which tables and codeunits to read next.

### Patterns that show up in BC profiles

- `FindSet` over a large table where `FindFirst` or `IsEmpty` would do
- Event subscriber on a hot path (e.g. `OnAfterInsertEvent` on a posting routine) that does extra DB reads
- Nested loops calling `Get` per iteration when a single query would suffice
- `CalcFields` of FlowFields inside a loop
- Missing `SetCurrentKey` on a filtered loop, forcing a table scan

## Hand off

When you send a profile to a partner:

- Describe the action in plain language: "Open Customer card for customer 30000, time from click to fully loaded."
- Include the exact reproduction steps you used during capture.
- State expected vs actual timing if you have it: "Used to take 2 seconds, now 8 seconds."
- Mention the BC version and the list of installed extensions (a screenshot of Extension Management is enough).

## What the profiler will NOT show

- Network latency between client and service tier
- SQL execution plans (it shows AL-side time, not the SQL engine's plan)
- Job Queue contention (capture during the slow user-facing operation, not the queue)

For those, telemetry (`appi-eql-prod-tenant`) and BC server-side telemetry are the right tool.

## Related skills

- `al-code-review` for the patterns that the profiler will surface

## References

- Microsoft docs on Performance Profiler: https://learn.microsoft.com/dynamics365/business-central/dev-itpro/performance/performance-troubleshooting#performance-profiler
