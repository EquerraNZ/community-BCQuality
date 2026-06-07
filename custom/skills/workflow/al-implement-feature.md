---
kind: task-skill
id: al-implement-feature
version: 1
title: Implement a feature from its spec
description: Implement a Business Central feature from its approved spec.md, plan.md, and tasks.md, applying the house rules and BCQuality, then run the mandatory verifier agents, fix findings, and update docs. Use as the final step of the Spec-Driven Development loop for a feature.
bc-version: [all]
technologies: [al]
countries: [w1]
application-area: [all]
---

# Implement Feature

Execute an approved plan. The spec is the brain; here the agent is the muscle. See
`AGENTS.md` for the full contract.

## Preconditions

`specs/features/<id>/spec.md`, `plan.md`, and `tasks.md` exist and are approved.
If not, run `/al-spec-feature` then `/al-plan-feature` first. Do not write production AL
without an approved plan.

## Steps

1. **Load the feature.** Read the spec, plan, and tasks for `<id>`, plus the
   constitution (`brief.md`, `tech-design.md`) and the house rules in
   `al-code-review`.
2. **Work the tasks in order.** Implement `tasks.md` top to bottom. For each AL
   object, follow the house rules: object IDs in the assigned range, `Label` and
   `Caption` with `Comment`, business logic in codeunits not triggers, telemetry
   on protected operations, no `ODataKeyFields`, permission-set entries for every
   new object. Tick each task as it lands.
3. **Write the tests** named in the plan, one or more per acceptance criterion in
   the spec. Use `ai-test-driven-development` for Copilot/agent features.
4. **Build.** Compile the app. The post-build hook will prompt the verifier pass.
5. **Verify.** Run the mandatory set in parallel: `al-code-quality-reviewer`,
   `al-readability-checker`, `al-test-coverage-validator`, `al-test-validator`,
   plus a BCQuality review. Add `al-performance-reviewer` for hot paths and
   `al-upgrade-checker` if the schema changed. Resolve every blocking finding,
   citing the BCQuality rule that drove each fix.
6. **Confirm acceptance.** Check each acceptance criterion in `spec.md` is covered
   by a passing test. Re-run the verifiers until clean.
7. **Docs and roadmap.** Update any feature docs (and run `bc-extension-test-guide`
   if a TEST_GUIDE is maintained), then set the roadmap item to `done`.
8. **Merge.** Prepare the PR. Reference the feature id and link the spec.

## Output

The implemented AL under the app folder, passing tests, a clean verifier pass with
BCQuality citations, updated docs, and a roadmap item marked done.

## Related

- The verifier agents and `bcquality-integration` (the citation contract).
- `al-appsource-validation` before an AppSource submission.
