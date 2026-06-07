---
kind: task-skill
id: plan-feature
version: 1
title: Produce a feature plan and task list
description: Produce the technical plan and ordered task list (plan.md + tasks.md) for a Business Central feature whose spec.md is approved. Maps the spec to AL objects, an object ID range, a data model, and verifiable tasks. Use after /spec-feature and before /implement-feature.
bc-version: [all]
technologies: [al]
countries: [w1]
application-area: [all]
---

# Plan Feature

Translate an approved feature spec into a concrete, reviewable build plan. This is
the bridge from what to how. No production AL is written yet. See `AGENTS.md`.

## Preconditions

`specs/features/<id>/spec.md` exists and is approved. If acceptance criteria or
open questions are unresolved, go back to `/spec-feature` first.

## Steps

1. **Read the inputs.** The feature `spec.md`, `specs/tech-design.md` (reuse
   standard BC, object ID range, data model), and the house rules in
   `.claude/skills/al-code-review/SKILL.md`.
2. **Design the implementation.** Decide which standard BC modules to reuse and
   what custom AL is genuinely needed. Every new object gets an ID inside the
   assigned range.
3. **Write `plan.md`** from `specs/templates/feature-plan.md`: approach, standard
   BC reused, the AL object table (name, type, id, new/extend, purpose), data
   model, integration points, cross-cutting concerns (permissions, telemetry,
   upgrade/migration, performance), risks, and the test strategy.
4. **Write `tasks.md`** from `specs/templates/feature-tasks.md`: an ordered,
   checkable list. Include explicit tasks for permission-set entries, telemetry,
   tests per acceptance criterion, the build-and-verify pass, and docs/roadmap.
5. **Pre-flight against the verifiers.** Sanity-check the plan against what the
   verifier agents will enforce later (object IDs in range, IDataAccess seam where
   logic touches the DB, no `ODataKeyFields`, upgrade step for any schema change).
   Adjust the plan so the Implement step starts clean.
6. **Update the roadmap** item status to `planned`.
7. **Stop for review.** Summarise the plan and the object list. Do not implement
   until the user approves.

## Output

`specs/features/<id>/plan.md` and `tasks.md`, plus an updated roadmap row.

## Related

- `al-code-review` (house rules), `performance-profiler`, `major-release-readiness`
  (if the plan changes schema), `appsource-validation` (if bound for AppSource).
- `/implement-feature` once the plan is approved.
