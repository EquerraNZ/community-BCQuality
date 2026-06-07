---
kind: task-skill
id: al-go-environment-onboarding
version: 1
title: Register a BC environment for AL-Go deployment
description: Register a customer Business Central sandbox or production environment for AL-Go deployment. Use when adding a new customer environment to an AL-Go repo, or troubleshooting a "deploy did not run" or "deployed version is wrong" situation.
bc-version: [all]
technologies: [al, powershell]
countries: [w1]
application-area: [all]
---

# AL-Go Environment Onboarding

End-to-end setup to wire a Business Central environment into an AL-Go repo so CI/CD can deploy to it. Covers Entra App Registration, BC user provisioning, GitHub environment with `AUTHCONTEXT`, and the AL-Go settings block.

## When to use

- Adding a new customer sandbox or production environment to an AL-Go repo
- Promoting an internal sandbox to a customer-facing one
- Diagnosing a Continuous Deployment run that did not fire, or fired but deployed nothing
- Validating that environment-specific versioning is set correctly

## Pre-requisites

- Admin access on the target AL-Go repo
- BC admin access to the customer tenant (or coordination with the customer)
- The shared Entra App Registration `BC-CICD-NonProd` (or its production equivalent)

## Step-by-step

### 1. Entra App Registration (S2S authentication)

Use the existing shared app registration where possible, do not create a new one per customer.

- App name: `BC-CICD-NonProd` (for sandbox) or production equivalent
- Auth type: Service-to-Service (S2S) using client credentials
- API permissions: `Dynamics 365 Business Central` > `app_access` (application permission), granted with admin consent

Capture:
- Tenant ID
- Client ID
- A fresh client secret (note the expiry, set a calendar reminder to rotate)

### 2. BC user setup for the App Registration

Create the corresponding BC user in both Production and Sandbox environments of the customer tenant.

- User type: Application
- Assign permission sets:
  - `D365 AUTOMATION`
  - `EXTEN. MGT. - ADMIN`
- In **Production**, disable the user. AL-Go CI/CD only deploys to non-production unless you explicitly opt in. Leaving the user enabled in production is an outage risk.
- In **Sandbox**, leave enabled.

### 3. GitHub environment

In the AL-Go repo: **Settings > Environments > New environment**.

- Name: **exactly** the BC environment name. The names must match character for character. If your BC sandbox is `Test-NZ`, the GitHub environment is `Test-NZ`.
- Add environment secret `AUTHCONTEXT` with a compressed JWT payload:

```json
{
  "tenantId": "<entra-tenant-guid>",
  "scopes": "https://api.businesscentral.dynamics.com/.default",
  "clientId": "<app-registration-client-id>",
  "clientSecret": "<app-registration-client-secret>"
}
```

Compress with no whitespace (no newlines) and paste as the secret value. Some teams base64 the JSON; AL-Go accepts both, follow whichever convention the rest of your AL-Go repos use.

### 4. AL-Go-Settings.json deploy block

In `.github/AL-Go-Settings.json`, add the environment to the `environments` array and create a `DeployTo<EnvName>` block:

```json
{
  "environments": ["Test-NZ"],
  "DeployToTest-NZ": {
    "EnvironmentType": "SaaS",
    "EnvironmentName": "Test-NZ",
    "Branches": ["main"],
    "SyncMode": "Add",
    "ContinuousDeployment": true,
    "runs-on": "windows-latest"
  }
}
```

Key fields:

| Field | Notes |
|---|---|
| `EnvironmentType` | `SaaS` for customer environments, `OnPrem` only when you have a runner on the customer network |
| `EnvironmentName` | Must match BC environment name AND the GitHub environment name |
| `Branches` | Which branches deploy to this environment. `main` for permanent sandbox, release branches for production |
| `SyncMode` | `Add` for additive deploys, `Clean` for full sync. Default `Add` unless schema changes require otherwise |
| `ContinuousDeployment` | `true` to deploy on every push that matches `Branches`, `false` for manual-only |

### 5. Versioning

Always define versioning explicitly in the AL-Go settings. Do not rely on AL-Go's inherited defaults: they have surprised teams (deploying older versions, or partial deploys).

Validate by running CI/CD on `main` once and confirming the deployed version on the BC environment matches `app.json` in the repo.

## Common failure modes

| Symptom | Cause | Fix |
|---|---|---|
| CD did not run on push | `Branches` in `DeployTo<Env>` does not include the pushed branch | Add the branch, run **Update AL-Go System Files** |
| CD ran but nothing deployed | `ContinuousDeployment: false` | Flip to `true` |
| Wrong version deployed | Relied on AL-Go inherited defaults | Define `appBuild`, `appRevision`, and version policy in settings explicitly |
| 401 from BC | App registration not consented, or BC user disabled in target env | Re-grant admin consent, enable the BC application user in the target env |
| Auth context invalid | JSON whitespace or missing field | Recompress, validate the four fields are present and the secret matches |

## Test continuous deployment early

Test CD on a throwaway sandbox before you wire up a customer's real environment. Confirms:
- The Entra app registration is consented
- The BC user has the right permission sets
- The GitHub environment name exactly matches BC
- The deploy block routes the right branch

## Related skills

- `al-go-pipelines` for the broader AL-Go framework
- `rbac-and-access` for managing the Entra app registration cleanly
- `security-group-setup` for restricting access on the deployed environment

## References

- AL-Go docs on environments: https://github.com/microsoft/AL-Go/blob/main/Scenarios/Setup.md
