---
kind: task-skill
id: al-spec-feature
version: 1
title: Write a feature specification
description: Write a feature specification (specs/features/<id>/spec.md) for one Business Central feature, grounded in the project constitution. Use before planning or implementing any feature. Produces requirements and acceptance criteria only, no AL.
bc-version: [all]
technologies: [al]
countries: [w1]
application-area: [all]
---

# Spec Feature

Turn a feature idea into a reviewable specification. The spec captures the what
and why; it does not name AL objects (that is `/al-plan-feature`). See `AGENTS.md`.

## Preconditions

The constitution must exist. If `specs/brief.md` or `specs/tech-design.md` is
missing or empty, run `/al-spec-init` first.

## Steps

1. **Read the constitution.** `specs/brief.md`, `specs/tech-design.md`,
   `specs/roadmap.md`. The spec must be consistent with all three.
2. **Choose the feature.** Use the next `todo` item on the roadmap, or the feature
   the user names. Confirm the feature id and slug (`NNN-slug`, matching the
   roadmap number).
3. **Clarify.** Ask the user about anything ambiguous: scope edges, rules, user
   roles, acceptance criteria. Record unresolved items under Open questions rather
   than guessing.
4. **Write the spec.** Create `specs/features/<id>/spec.md` from
   `specs/templates/feature-spec.md`: problem, users and roles, scope, out of
   scope, user flow, acceptance criteria (testable), data and rules, telemetry,
   open questions. Acceptance criteria must be concrete enough to become tests.
5. **Update the roadmap.** Set the item status to `spec`.
6. **Stop for review.** Summarise the spec and list open questions. Do not plan or
   write code until the user approves the spec.

## Output

`specs/features/<id>/spec.md` and an updated roadmap row. No AL, no object names.

## Related

- `/al-plan-feature` once the spec is approved.
- `bc-integrations`, `copilot-promptdialog`, `ai-agent-sdk` for scoping
  feature-specific behaviour when relevant.
