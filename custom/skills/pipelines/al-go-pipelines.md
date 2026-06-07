---
kind: task-skill
id: al-go-pipelines
version: 1
title: Run BC CI/CD with AL-Go for GitHub
description: How to run Business Central CI/CD with AL-Go for GitHub, including AL MCP Server integration for headless build and compile. Use when setting up, configuring, or troubleshooting an AL-Go repo, planning a release, or wiring `altool launchmcpserver` into CI.
bc-version: [all]
technologies: [al, powershell]
countries: [w1]
application-area: [all]
---

# AL-Go for GitHub Pipelines

The chosen framework for BC CI/CD on GitHub Actions. Maintained upstream by `@freddydk`. The strategic goal is to migrate every BC pipeline from Azure DevOps to GitHub Actions running AL-Go.

## When to use

- Setting up a new BC app repo for CI/CD
- Customising AL-Go behaviour (build, test, deploy, sign)
- Cutting a release on an AL-Go repo
- Diagnosing a failed AL-Go workflow run
- Reviewing a PR that touches `.AL-Go/settings.json`, `.github/AL-Go-Settings.json`, or the AL-Go workflow files

## Hard rules

1. **Never modify AL-Go workflow scripts directly.** The workflows are downloaded at runtime from the upstream AL-Go repo. Editing them in your repo breaks the auto-upgrade path. Customise via settings, not via patches.
2. **Releases use semantic version tags** (`v1.2.0`), are marked as **Pre-release**, and use **Create Release Branch** so a `release/v1.2.0` head exists from the moment of release.
3. **One repo per AppSource app or customer PTE app**, with multi-project repos allowed only when apps are tightly coupled.
4. **`main` is always the latest code.** Older versions live in tags. Never roll back `main`.

## Configuration model

Two settings files control everything:

| File | Scope | Examples of what lives here |
|---|---|---|
| `.AL-Go/settings.json` | Per project (build and test) | `projects`, `appFolders`, container country, AppSourceCop config, `appDependencyProbingPaths` |
| `.github/AL-Go-Settings.json` | Per repo (CI/CD wiring) | PR trigger branches, CI/CD trigger branches, `environments` array, `ContinuousDeployment`, dependent apps |

Most settings changes need the **Update AL-Go System Files** workflow to apply afterwards, because they affect the workflow scripts AL-Go pulls down.

## Repo and branching strategy

- **One repository per AppSource app or customer PTE app**. Related apps may share a repo using AL-Go's multi-project layout.
- **Default branch is `main`**, always latest code. Older releases are reachable via Git tags.
- **Release branches** are independent heads cut from `main`'s current state at the moment of release. They are not long-lived forks. A hotfix on a release branch is a `hotfix/<description>` working branch off the release branch.
- **Shared assets and config** live in a separate `d365-dependent-artifacts` repo, linked as a Git submodule. It holds signed apps today and shared scripts/docs/config later.

## d365-dependent-artifacts

Used as a Git submodule by AL-Go consumer repos.

- **Never make public**, never share externally
- `main` is protected, PR required, direct commits disallowed
- No CI/CD by design (the repo only stores artefacts, it does not build them)
- Org admins manage write access

## How to cut a release

1. Trigger the **Create Release** workflow.
   - First release of an app: trigger from `main`.
   - Subsequent releases: trigger from the most recent release branch.
2. Tag using semantic version (`v1.2.0`).
3. Mark as **Pre-release** until you have validated it.
4. Tick **Create Release Branch** so the `release/v1.2.0` branch is cut.
5. Apply hotfixes via `hotfix/<description>` branches off the release branch.
6. Bring main changes into the pre-release branch by pulling all of `main` (trunk-based) or cherry-picking. Open the PR back into the pre-release branch.
7. Promote to a full release only when validated.

## Update AL-Go System Files (one-time auth setup)

AL-Go needs to rewrite its own workflows to upgrade itself. That requires a GitHub App with write access to the repo, surfaced as the `GHTOKENWORKFLOW` repo secret.

1. Install the shared AL-Go GitHub App on the repo (reuse the org's shared app, which has no expiry). You need Admin on the repo to install.
2. Create the repo secret `GHTOKENWORKFLOW`. The value lives in the team's secret management tool under `GHTOKENWORKFLOW - AL GO`.
3. Run the **Update AL-Go System Files** workflow.
4. Review and merge the auto-generated PR that updates the workflow files.

## What NOT to do

- Do not hand-edit any file under `.github/workflows/AL-Go-*`. Those are owned by AL-Go.
- Do not skip the **Update AL-Go System Files** step after changing settings that affect workflow shape.
- Do not branch a release into `release/extension/NextMajor` until all customer environments are on the new major. See `major-release-readiness` for the upgrade governance.
- Do not commit signed apps into your AL-Go repo. They go into `d365-dependent-artifacts` and are pulled in via the submodule.

## AL MCP Server in CI

For agent-driven CI workflows that don't run inside VS Code, the AL MCP Server (`altool launchmcpserver`) gives any MCP-compatible agent the same tools AL-Go uses internally: build, compile, publish, symbol download, symbol search.

Use cases:

- A separate PR-gate workflow that runs `al_compile --onlyErrors` for fast pass/fail feedback before the full AL-Go build kicks in
- A chat-driven CI workflow where a developer asks Claude or Copilot Studio to "build and publish to UAT"
- Headless agents that orchestrate multi-repo BC pipelines

Pattern for a CI gate:

```bash
# 1. Start the AL MCP server in the runner
altool launchmcpserver --transport stdio &

# 2. The agent (or a wrapper script) sends:
#    al_auth_login -> al_downloadsymbols (globalSourcesOnly=true) -> al_compile (onlyErrors=true)

# 3. Exit 0/1 based on al_compile result.
```

Use `globalSourcesOnly: true` on `al_downloadsymbols` for CI: no BC connection or auth required, only Microsoft NuGet feeds and AppSource.

Keep AL-Go's own workflows in charge of the canonical build/test/publish path. The AL MCP server is for agent-driven or PR-gate use cases that AL-Go does not cover directly.

See `al-mcp-server` for the full tool reference and JSON-RPC envelope.

## Related skills

- `al-mcp-server` for the standalone AL MCP server tools used in agent-driven CI
- `al-go-environment-onboarding` for the per-environment setup (S2S, AUTHCONTEXT, deploy block).
- `major-release-readiness` for the rules around BC major version upgrades.

## References

- AL-Go for GitHub: https://github.com/microsoft/AL-Go
