---
kind: task-skill
id: bc-extension-test-guide
version: 1
title: Generate a BC extension test guide
description: Generate an exhaustive TEST_GUIDE.md for a Business Central AL extension. Inventories every page, field, TableRelation, enum, action, state machine, permission set, and telemetry event in the codebase and produces a category-driven release-audit checklist that covers the state pivots happy-path testing misses. Use after a feature lands, before a release, or when onboarding QA on an existing extension.
bc-version: [all]
technologies: [al]
countries: [w1]
application-area: [all]
---

# BC Extension Test Guide Generator

Produce `DOCS/TEST_GUIDE.md` for a Business Central AL extension. The guide is **exhaustive by construction**: every single field, relation, action, and reachable data state in the extension's AL source must appear in at least one category inventory. Happy-path coverage is not enough; this guide catches the state pivots that ship-then-regret bugs hide behind.

## When to use

- A new feature has landed and we need to refresh the QA audit.
- Before a release, to produce the checklist QA actually runs.
- Onboarding a new extension that has a `USER_GUIDE.md` but no QA companion.
- After a "we missed that bug" retro, to make sure the missed category is now inventoried.

Do not use this skill for:

- Writing AL unit tests (use `ai-test-driven-development` for Copilot/agent tests; standard BC test codeunit patterns otherwise).
- Persona-driven scenario simulation (different shape; lives in `persona-driven-e2e-testframework` if present in the toolkit).
- A single bug repro (just write the test directly).

## Workflow this skill belongs to

```
1. Skill (this one) → generates DOCS/TEST_GUIDE.md from AL source.
2. Agent (bc-webclient-runner or equivalent) → executes every TEST_GUIDE entry
   end-to-end against the deployed extension via Chrome MCP.
3. Findings → discussed with developer, fixed, redeployed.
4. Agent re-runs the affected TEST_GUIDE entries.
5. Release passes when every applicable category is PASS.
```

Generating the guide is step 1. Steps 2–5 are run by other agents/skills.

## Inputs

- Path to the extension repo root (must contain `app.json` and AL `src/`).
- Optional: existing `DOCS/USER_GUIDE.md` to cross-reference feature names.
- Optional: existing `DOCS/TEST_GUIDE.md` to diff against (this skill rewrites or appends as instructed).

## Output

`DOCS/TEST_GUIDE.md` populated with the 12 categories below. Every category includes a populated inventory drawn from the actual AL source, not placeholders.

## The 12 categories

This list is the contract. Every generated guide has exactly these 12 categories in this order.

| # | Category | What it catches |
|---|---|---|
| 1 | Lookup audit (happy + inline-create) | Noisy `QuickEntry`, mandatory PK asterisks, missing inline-create coverage |
| 2 | Type-conditional TableRelation | Lookup target swap by sibling type/subtype, broken per-type validation |
| 3 | Eligibility filters on lookups | Pick-then-error: lookup lists records that downstream code rejects |
| 4 | Visibility / Editable conditionals | UI state vars that don't refresh on the pivot they depend on |
| 5 | StandardDialog Mode pivots | Each Mode tested end-to-end including the commit, not just open |
| 6 | Subpage FK persistence | `SubPageLink` not propagating PK to inserted rows |
| 7 | State machine transitions | Every allowed transition + every disallowed rejection |
| 8 | Permission boundaries | Each permission set's RIMD claims actually hold |
| 9 | Telemetry events | Each documented event fires with the documented payload |
| 10 | Mobile / tablet smoke | Pages render on 375×812 viewport without horizontal scroll |
| 11 | Cross-company isolation | `DataPerCompany` correctness, no singleton leak |
| 12 | Upgrade paths | Schema changes preserve data version-to-version |

## Exhaustiveness contract

The output guide is invalid if **any** of the following is missing.

### Category 1 (Lookup audit) must list

- Every field on every user-facing page where the underlying table field has `TableRelation = ...` to another table.
- Both directions: source field (page + field) → target table + Card page.
- Cross-checked against `grep -rn "TableRelation" src/`.

### Category 2 (Type-conditional TableRelation) must list

- Every field whose `TableRelation` is conditional on another field on the same record (uses `where(...)` referencing a sibling field, or whose `OnLookup` / `OnValidate` branches on a sibling).
- Every enum/option value of the sibling selector, mapped to its target table.

### Category 3 (Eligibility filters) must list

- Every lookup whose downstream `OnValidate` or OK handler errors on a subset of the target table (e.g. status-blocked records, `Blocked = true`).
- For each: the rejected predicate, and whether the lookup currently pre-filters it.

### Category 4 (Visibility / Editable conditionals) must list

- Every `Visible = <pageVar>` and `Editable = <pageVar>` on every page.
- Every driver (page var, `Rec.<field>`, `CurrPage.Editable`) that determines those flags.
- Whether the driver's `OnValidate` calls `CurrPage.Update(false)` (or equivalent) to force the visual refresh.

### Category 5 (StandardDialog Mode pivots) must list

- Every page with `PageType = StandardDialog` that has an `Option` or enum `Mode` selector driving conditional fields.
- Each Mode value, the visible-field set per Mode, the documented OK side effect per Mode, the documented error per Mode on blank required input.

### Category 6 (Subpage FK persistence) must list

- Every `part(...)` on a Card or Document page that uses `SubPageLink = ...`.
- The FK fields propagated.
- Whether `SetParentKeys()` explicit-push pattern is in use or whether the subpage relies on `SubPageLink` default-value behaviour alone (the latter is a known fragility point, flag it).

### Category 7 (State machine transitions) must list

- Every `enum` used as a `Status` field on a table.
- For each: the full from×to matrix, marking allowed vs. disallowed with the trigger that causes the transition.
- Every code path that mutates the Status field (search for `Status :=` or `Validate("Status",`).

### Category 8 (Permission boundaries) must list

- Every `permissionset` object in the extension.
- For each permission set: every table-data permission claim (R/I/M/D), every page/codeunit permission claim.
- Cross-referenced against the actual tables/pages in the extension.

### Category 9 (Telemetry events) must list

- Every call to a telemetry codeunit that logs a custom event.
- Event ID, trigger, expected payload keys.
- Cross-checked against `grep -rn "Telemetry.LogEvent\|Session.LogMessage" src/`.

### Category 10 (Mobile / tablet smoke) must list

- Every top-level user-facing page (`UsageCategory <> None` or reachable from a top-level page's actions/parts).

### Category 11 (Cross-company isolation) must list

- Every table in the extension.
- Each table's `DataPerCompany` value (default true if unspecified).
- Any singleton or shared-state table called out explicitly.

### Category 12 (Upgrade paths) must list

- Every upgrade codeunit (`Subtype = Upgrade`).
- Every documented schema change in the current release notes vs. the previous shipped version.
- Test seed instructions for each schema change.

## Procedure

Execute these steps in order. Do not skip discovery to save time; the value of this skill is the exhaustiveness of the inventory.

### Step 1. Discover the repo

```
- Read app.json: id, name, version, dependencies, idRanges.
- Read .AL-Go/settings.json: appFolders, testFolders.
- Identify the primary app folder (usually <Name>.Main/).
- glob src/**/*.al → full file list.
```

### Step 2. Build the field inventory

For each `*.Table.al`:

- List every field: ID, name, type, `TableRelation`, `NotBlank`, triggers (`OnValidate`, `OnLookup`, `OnInsert`, `OnModify`, `OnDelete`).
- Note `DataPerCompany`, primary key, secondary keys, field groups.
- Note any `FieldClass = FlowField` and the `CalcFormula`.

### Step 3. Build the page inventory

For each `*.Page.al` and `*.PageExt.al`:

- `PageType`, `SourceTable`, `UsageCategory`, `ApplicationArea`.
- Every field: bound `Rec.<field>` or page var, `Visible`, `Editable`, `ShowMandatory`, `QuickEntry`, `TableRelation` (if overridden at page level).
- Every action: trigger code, `RunObject`, `RunPageLink`, `Promoted`, `Image`.
- Every `part(...)` and its `SubPageLink`.
- Every page-level trigger: `OnOpenPage`, `OnAfterGetCurrRecord`, `OnQueryClosePage`, `OnNewRecord`, `OnInsertRecord`.

### Step 4. Build the codeunit / enum / permission set inventory

- Enums: every value, `Extensible` flag.
- Permission sets: full RIMD per table, page execute claims, indirect permissions.
- Codeunits: identify upgrade codeunits, event subscribers (especially OnBefore/AfterDelete on standard tables), telemetry call sites.

### Step 5. Populate the categories

Map every inventoried item to its category(ies). One item can appear in multiple categories (e.g. an Origin Code field appears in Category 1 lookup audit AND Category 2 type-conditional AND Category 3 eligibility filter if `Blocked` is checked).

A common failure mode: stopping when you've found "enough" entries. Don't. Every TableRelation gets a row in Category 1, no exceptions. Every page-level Visible flag gets a row in Category 4. Every enum-as-status gets a transition matrix in Category 7. If a category's inventory has fewer rows than the underlying AL would warrant, the guide is incomplete.

### Step 6. Write `DOCS/TEST_GUIDE.md`

Use the template below. Keep the category numbering and order stable across regenerations so diffs across releases are meaningful.

### Step 7. Self-audit pass

Before declaring done, run:

- `grep -rn "TableRelation" src/` → every match must appear in Category 1 or 2.
- `grep -rn "Visible = \|Editable = " src/**/*.Page.al` → every dynamic match must appear in Category 4.
- `grep -rn "PageType = StandardDialog" src/` → every match must appear in Category 5 (if it has a Mode), Category 1 if it has lookups, etc.
- `grep -rn "SubPageLink" src/` → every match must appear in Category 6.
- `grep -rn "permissionset" src/` → every match must appear in Category 8.
- `grep -rn "LogEvent\|LogMessage" src/` → every match must appear in Category 9.

If any of these grep counts exceed the inventory rows, the guide is incomplete. Add the missing entries. Repeat until all greps match.

## Template

Use this exact structure when writing `DOCS/TEST_GUIDE.md`. Fill every inventory table with rows drawn from the actual AL.

```markdown
# <Extension Name>, Test Guide

The QA companion to USER_GUIDE.md. Organized by category, not by feature.
Each category catches a class of bug that happy-path testing misses.

## 0. Category index
<the 12-row table>

## 1. Lookup audit (happy + inline-create)
<definition, procedure, fix pattern, inventory table>

## 2. Type-conditional TableRelation
<...>

## 3. Eligibility filters on lookups
<...>

## 4. Visibility / Editable conditionals
<...>

## 5. StandardDialog Mode pivots
<...>

## 6. Subpage FK persistence
<...>

## 7. State machine transitions
<...>

## 8. Permission boundaries
<...>

## 9. Telemetry events
<...>

## 10. Mobile / tablet smoke
<...>

## 11. Cross-company isolation
<...>

## 12. Upgrade paths
<...>

## Run report template
<rolling release-pass scorecard>
```

A worked example is shipped at `examples/elevate-shipping-TEST_GUIDE.md` in this skill folder. Refer to it for the exact level of inventory granularity expected. Do not copy it as a template, the worked example is for *that* extension; the structure transfers but every row in every inventory must come from the target extension's AL.

## Output checklist

Before signaling done:

- [ ] All 12 categories present, in order, with definition + procedure + inventory.
- [ ] Every `TableRelation` in `src/` accounted for in Cat 1 or Cat 2.
- [ ] Every dynamic `Visible`/`Editable` accounted for in Cat 4.
- [ ] Every `PageType = StandardDialog` with a Mode accounted for in Cat 5.
- [ ] Every `SubPageLink` accounted for in Cat 6.
- [ ] Every status `enum` with a transition matrix in Cat 7.
- [ ] Every `permissionset` with full RIMD in Cat 8.
- [ ] Every telemetry call site in Cat 9.
- [ ] Every top-level user-facing page in Cat 10.
- [ ] Every table's `DataPerCompany` in Cat 11.
- [ ] Every upgrade codeunit + per-release schema delta in Cat 12.
- [ ] Run report template at the end.
- [ ] No "TODO" or "n/a" placeholders in inventory tables (an empty inventory means the category genuinely doesn't apply; say so explicitly with one line, do not leave a stub).
- [ ] Cross-link added in USER_GUIDE.md header pointing to TEST_GUIDE.md.

## Non-goals

- This skill does not run tests. It only produces the artifact tests run against.
- This skill does not infer test data shapes from BC standard. Seed data is a separate concern handled per-test by the runner agent.
- This skill does not write AL test codeunits. Use the standard BC test codeunit pattern (or `ai-test-driven-development` for Copilot/agent regression).
- This skill is not a replacement for unit tests. It is a release-pass exhaustiveness contract.
