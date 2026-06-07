---
kind: task-skill
id: al-major-release-governance
version: 1
title: Govern a BC major version upgrade
description: Governance for Business Central major version upgrades. Use when planning compatibility testing for a new BC major, deciding whether to cut a NextMajor branch, or reviewing a PR that bumps `app.json` application or platform versions.
bc-version: [all]
technologies: [al]
countries: [w1]
application-area: [all]
---

# Major Release Readiness

How to manage customer-specific and shared extensions across BC major version upgrades. The big idea is: a NextMajor branch is expensive (parallel maintenance), so we delay it as long as the current Production version still compiles and runs cleanly.

## When to use

- A new BC major version (e.g. BC 27) is approaching general availability
- Reviewing a PR that changes `app.json` `application` or `platform` minimum versions
- Deciding whether to create a `release/extension/NextMajor` branch
- Triaging a deprecation warning on the current version

## Core rules

### For customer extensions

1. **Do not create `release/extension/NextMajor`** until **all customer environments** running that extension have been upgraded to the latest major.
2. Test compatibility on the next major **without changing `app.json` `application` or `platform` versions**. Compatibility testing is a check, not a commitment.
3. Resolve all **warning** and **info** messages on the current version. Defer warnings that would require taking a dependency on NextMajor features.
4. Microsoft gives **at least one year notice** before deprecations are removed. Deferral is safe; rushing into NextMajor is not.
5. Verify the extension still compiles on the current Production version after any NextMajor exploration.

### For shared extensions (multi-customer)

1. Branch `release/<Extension>/NextMajor` **only when** all SaaS customers running the extension are on the latest major OR a specific feature mandates the new platform.
2. The extension owner is responsible for creating and maintaining the NextMajor branch. Delegation requires explicit agreement.
3. Mirror continuous improvement changes into `Sandbox-NextMajor` regularly. Testing reflects latest code, not a snapshot.
4. For non-breaking changes that work on both versions, **refactor on the current version first**. Only branch NextMajor if the change cannot work on the older version.

## Workflow when a new BC major lands

```
1. Spin up a Sandbox-NextMajor environment for the extension.
2. Install the current Production version of the extension on it.
3. Run regression tests. Capture warnings and errors.
4. Triage each warning:
   - Resolvable on current Production version? Fix forward, deploy as usual.
   - Requires dependency on NextMajor? Defer. Track in backlog. Do NOT bump app.json.
   - Hard breaking change? Document in the NextMajor planning doc, do not cut the branch yet.
5. Mirror new continuous-improvement commits from main into Sandbox-NextMajor on a regular cadence.
6. Once all customers are on the new major OR a feature mandates it, the extension owner cuts release/<Extension>/NextMajor.
```

## When you must cut NextMajor early

A feature that cannot be implemented on the current major is the only good reason. Examples:
- New API surface only available on NextMajor
- Schema change Microsoft requires for compatibility
- Performance feature (e.g. new query type) needed for customer SLA

In those cases:
- Document the specific feature driving the early branch in the PR description
- Add a follow-up task to backport equivalents to the current major if at all possible
- Communicate to customers on older majors that this branch is unavailable to them until they upgrade

## What "ready to upgrade customer" means

A customer is ready for the new BC major when:

1. Every shared extension installed on their tenant has a tested NextMajor build (or is confirmed clean on current version).
2. Their custom (PTE) extensions have been compatibility-tested.
3. Sandbox-NextMajor has run a full regression for at least one full release cycle.
4. The customer has agreed to the upgrade window.

Do not push customers to upgrade just to allow the team to drop the old major. Microsoft's deprecation window is long enough to plan properly.

## Related skills

- `al-go-pipelines` for the release-branch mechanics
- `al-code-review` for AL changes that touch deprecated APIs

## References

- Microsoft BC release schedule: https://learn.microsoft.com/dynamics365/business-central/dev-itpro/upgrade/upgrade-overview-v15
