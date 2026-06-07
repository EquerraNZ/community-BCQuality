---
kind: task-skill
id: bcquality-integration
version: 1
title: Consume the BCQuality knowledge corpus
description: How the verifier agents consume Microsoft's BCQuality knowledge corpus (EquerraNZ/community-BCQuality) as a source of citable BC quality rules. Use when a verifier agent needs to cite a Microsoft-vetted rule, or when authoring a new agent that emits findings against BC code.
bc-version: [all]
technologies: [al]
countries: [w1]
application-area: [all]
---

# BCQuality Integration

How the verifier agents (`al-code-quality-reviewer`, `al-readability-checker`, `al-test-validator`, `al-test-coverage-validator`) consume Microsoft's [BCQuality](https://github.com/EquerraNZ/community-BCQuality) knowledge corpus.

## What BCQuality is

BCQuality is a Microsoft-maintained knowledge corpus for BC quality. It ships:

- **Meta-skills** at `bcquality/skills/`: `entry.md` (the dispatch entry point), `do.md` (output contract), `read.md` (knowledge file format), `write.md` (authoring guide).
- **Microsoft-authored content** at `bcquality/microsoft/`:
  - `skills/review/` ships seven review skills (`al-code-review`, `al-performance-review`, `al-privacy-review`, `al-security-review`, `al-style-review`, `al-ui-review`, `al-upgrade-review`).
  - `knowledge/` ships hundreds of atomic, citable knowledge files grouped by area (`performance/`, `privacy/`, `security/`, `style/`, `testing/`, `ui/`, `upgrade/`). Each rule is `<rule>.md` plus paired `<rule>.bad.al` and `<rule>.good.al` examples.
- **Partner layers** `community/` and `custom/`, populated in this fork: `community/` adds community-contributed performance and security rules, and `custom/` adds project-specific knowledge (api, integration, operations, performance, process) plus the review and testing skills.

The architecture is documented in `bcquality/agent-consumption.md`.

## How BCQuality is included

A Microsoft-authored subset of [BCQuality](https://github.com/EquerraNZ/community-BCQuality) is vendored directly into this repo as plain files under `.claude/bcquality/`. There is no submodule to initialise: the files are committed, so the agents can reference rules out of the box.

## Vendored layout

```
<project>/
  .claude/
    bcquality/
      skills/
        entry.md
        do.md
        read.md
        write.md
        README.md
      microsoft/
        skills/review/
          al-code-review.md
          al-performance-review.md
          al-privacy-review.md
          al-security-review.md
          al-style-review.md
          al-ui-review.md
          al-upgrade-review.md
        knowledge/
          performance/...
          privacy/...
          security/...
          style/...
          testing/...
          ui/...
          upgrade/...
      LICENSE        (MIT)
      README.md
      agent-consumption.md
```

The community/ and custom/ layers are not vendored (empty upstream). If Microsoft populates them in future, re-vendor from upstream to pick them up.

## How the agents use BCQuality knowledge

Each verifier agent has a **Knowledge sources** section in its system prompt that names the relevant BCQuality folder. For example:

| Agent | Primary BCQuality folder(s) |
|---|---|
| `al-code-quality-reviewer` | `microsoft/knowledge/performance/`, `microsoft/knowledge/security/`, `microsoft/knowledge/privacy/` |
| `al-readability-checker` | `microsoft/knowledge/style/`, `microsoft/knowledge/ui/` |
| `al-performance-reviewer` | `microsoft/knowledge/performance/` |
| `al-upgrade-checker` | `microsoft/knowledge/upgrade/` |
| `al-test-validator` | `microsoft/knowledge/testing/` |
| `al-test-coverage-validator` | none (coverage is structural; the agent supports the `references[]` field for forward-compatibility but does not cite knowledge by default) |

Agents without a one-to-one BCQuality domain (`al-appsource-validator`, `al-multitenancy-reviewer`, `al-translation-auditor`, `al-permission-set-auditor`, `al-obsolete-tracker`, `al-event-subscriber-auditor`) report against their own rule ids with `references: []` until matching knowledge files land upstream.

When the agent emits a finding that maps onto a BCQuality knowledge file, it includes a `references[]` entry in the output JSON:

```json
{
  "passed": false,
  "blocks": [
    {
      "rule": "avoid-commit-inside-loops",
      "file": "src/EventPostingMgt.al",
      "line": 88,
      "detail": "Commit() inside a foreach loop. Move outside or split the loop.",
      "references": [
        {
          "path": ".claude/bcquality/microsoft/knowledge/performance/avoid-commit-inside-loops.md",
          "sha": "35e02c2"
        }
      ]
    }
  ]
}
```

The `path` is relative to the consumer project root. The `sha` is the BCQuality commit pinned in the lock file (also exposed via `lock.bcquality.sha`). Tools that render agent findings (Claude Code UI, CI annotations, PR comments) can use the path to link the reviewer to the rule and the SHA to verify provenance.

## The cite-over-paraphrase rule

When an agent finding maps onto an existing BCQuality knowledge file, the agent **must** cite it via `references[]` rather than paraphrasing the rule from memory. Two reasons:

1. **Audit trail**. The reviewer (human or agent) can open the rule, see the bad/good AL examples, and verify the agent's interpretation.
2. **Update propagation**. When Microsoft updates a rule, refreshing BCQuality changes the rule content. Cited findings inherit the update automatically; paraphrased findings go stale silently.

Agents may add commentary alongside a citation (e.g. project-specific severity, suggested fix in the project's idiom), but the citation is mandatory when one exists.

## House rules that BCQuality does not cover

BCQuality is Microsoft-vetted neutral BC quality. Project-specific rules (no `ODataKeyFields`, the project's object ID range, the project's telemetry pattern, specific subscriber naming, etc.) live in the `al-code-review` skill and are applied **in addition** to BCQuality. Findings on project-specific rules carry an empty `references: []` and use a rule slug prefixed `house:` (e.g. `house:no-odata-key-fields`).

## Refreshing BCQuality

Re-vendor the subset under `.claude/bcquality/` from the latest upstream [EquerraNZ/community-BCQuality](https://github.com/EquerraNZ/community-BCQuality), then commit the updated files:

```bash
git add .claude/bcquality
git commit -m "chore: bump vendored BCQuality to <new-sha>"
```

Review the diff before merging, since rule-content changes can affect agent findings.

## License

BCQuality is MIT-licensed (Microsoft Corporation). The `LICENSE` file is vendored alongside the content. No attribution beyond that file is required for use; this skill credits Microsoft as the upstream author.

## See also

- BCQuality repository: https://github.com/EquerraNZ/community-BCQuality
- BCQuality agent consumption guide: `.claude/bcquality/agent-consumption.md` (in consumer projects)
- Agents that cite BCQuality: `agents/al-code-quality-reviewer.md`, `agents/al-readability-checker.md`, `agents/al-test-validator.md`, `agents/al-test-coverage-validator.md`
