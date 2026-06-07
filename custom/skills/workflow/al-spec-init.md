---
kind: task-skill
id: al-spec-init
version: 1
title: Scaffold the SDD constitution
description: Scaffold or refresh the Spec-Driven Development constitution (specs/brief.md, specs/tech-design.md, specs/roadmap.md) for a Business Central solution. Use once at the start of a project, or when the high-level business need changes. Run before writing any feature spec.
bc-version: [all]
technologies: [al]
countries: [w1]
application-area: [all]
---

# Spec Init

Establish the project constitution: the durable, high-level documents every agent
reads before doing anything. The spec is the brain; this is where the brain
starts. See `AGENTS.md` for the full workflow.

## When to use

- A new solution with no `specs/brief.md` yet.
- The customer requirements or technical strategy changed materially.

## Steps

1. **Read what exists.** If `specs/brief.md`, `specs/tech-design.md`, or
   `specs/roadmap.md` already have content, treat this as a refresh, not a
   rewrite. Preserve decisions still valid.
2. **Interview.** Ask the user for anything missing: customer and localisation,
   the business processes, goals and non-goals, constraints, the assigned object
   ID range, and which standard BC modules are in play. Do not invent facts.
3. **Write `specs/brief.md`** from `specs/templates` guidance: customer/context,
   goals, non-goals, key business processes, constraints, success measures. Plain
   language, no AL.
4. **Write `specs/tech-design.md`:** architecture overview, standard BC modules to
   reuse, the honest custom-code gaps, object ID range, high-level data model,
   integrations, cross-cutting concerns. Reuse standard BC first; justify every
   custom-code gap.
5. **Write `specs/roadmap.md`:** an ordered, numbered feature list with status
   `todo`. Number features so folders match (`001-...`, `002-...`).
6. **Stop for review.** The constitution is a human decision. Summarise what you
   wrote and the open questions; do not proceed to `/al-spec-feature` until the user
   confirms.

## Output

The three constitution files under `specs/`, plus a short summary of decisions
made and questions still open. No AL is written in this step.

## Related

- `/al-spec-feature` to specify the first roadmap feature once the constitution is approved.
- `al-code-review` for the house rules the technical design must respect.
