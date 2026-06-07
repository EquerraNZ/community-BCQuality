---
kind: task-skill
id: rbac-and-access
version: 1
title: Apply the Azure RBAC model
description: Azure RBAC model. Use when granting access to a workload, choosing a role, deciding on Managed Identity vs connection strings, or auditing who has Owner on a resource group.
bc-version: [all]
technologies: [powershell]
countries: [w1]
application-area: [all]
---

# Azure RBAC and Access

How to run Azure role-based access control. Least privilege at the resource group level, Managed Identity for runtime auth, no Owner role unless absolutely necessary.

## When to use

- Granting access to an Azure workload to a new team member
- Wiring two Azure services together (Function → Key Vault, Logic App → Service Bus)
- Reviewing or rotating role assignments
- Adding access to Key Vault for a new consumer
- Deciding whether a Managed Identity or a connection string is right

## Core principles

1. **Least privilege**. Grant the minimum role needed for the job. Read-only when read-only will do.
2. **Scope at the resource group**, not the individual resource. Resource group is the unit of workload. Same scope for access.
3. **Managed Identity for runtime auth**. Resources authenticate to each other using Managed Identity, not stored connection strings or secrets, wherever possible.
4. **Key Vault uses RBAC, not access policies**. The legacy access policy model is deprecated for new workloads.
5. **Owner role is reserved**. Contributor handles almost everything. Owner is only granted when role assignments themselves need to be managed.
6. **Quarterly review**. Calendar a 30-minute review of role assignments per RG. Stale access is real risk.

## Role assignments per resource group

| Resource Group | Role | Assignment Type | Notes |
|---|---|---|---|
| `rg-eql-prod-subscription` | Contributor | Platform team (group) | Subscription workflow ops |
| `rg-eql-prod-subscription` | Key Vault Secrets User | Subscription Logic Apps (System-Assigned MI) | Read secrets at runtime |
| `rg-eql-prod-subscription` | Key Vault Administrator | Platform team (group) | Manage secrets |
| `rg-eql-prod-integration` | Contributor | Platform team (group) | Integration workflow ops |
| `rg-eql-prod-integration` | Service Bus Data Owner | Integration components (MI) | Pub/sub messaging |
| `rg-eql-prod-telemetry` | Contributor | Platform team (group) | Telemetry workflow ops |
| `rg-eql-prod-telemetry` | Monitoring Metrics Publisher | Tenant telemetry publishers (MI) | Publish to AI |
| `rg-eql-prod-shared` | Contributor | Platform team (group) | Shared infra ops |
| `rg-eql-prod-shared` | Reader | Support team (group) | Read logs and metrics for triage |

Role assignments below resource group level (single resource) are an exception, not the default. If you find yourself doing them, ask why the resource doesn't belong in its own RG.

## Managed Identity rules

- **Use System-Assigned Managed Identity by default.** Lifecycle is tied to the resource. Deleting the resource cleans up the identity.
- **Use User-Assigned Managed Identity** when multiple resources need to share an identity (e.g. a fleet of Functions all hitting the same Key Vault).
- **Grant role assignments to the MI** at the resource group of the target service, not the source.
  Example: a Logic App in `rg-eql-prod-subscription` needs to read secrets from `kv-eql-prod-subscription`. Assign `Key Vault Secrets User` to the Logic App's MI at the Key Vault.
- **Cross-RG access works the same way**. Assign the MI a role at the target RG. No connection strings, no firewall holes.

## Key Vault specifics

- New Key Vaults use **RBAC**, not access policies. Set "Permission model" to RBAC at creation.
- Two roles cover most cases:
  - `Key Vault Secrets User` for consumers (read secrets)
  - `Key Vault Administrator` for the platform team (manage secrets)
- Use `Key Vault Secrets Officer` for service principals that need to **write** secrets (e.g. CI/CD that rotates).
- Never grant Reader at the Key Vault level expecting it to read secrets. Reader only sees the Key Vault metadata, not the secrets themselves.

## What not to do

- **No Owner at RG level** unless the role assignment management itself needs to be delegated. Contributor handles almost everything.
- **No custom roles** unless built-in roles are genuinely too broad. Custom roles drift, built-ins don't.
- **No persistent role assignments for elevated access**. Use **PIM (Privileged Identity Management)** to elevate just-in-time.
- **No connection strings in code**. Use Managed Identity. If you must use a key, store it in Key Vault and read via MI.
- **No "I just need this for 5 minutes" lingering assignments**. Document, deadline, and remove.

## Privileged Identity Management (PIM)

For roles that engineers need occasionally but not always:

- Eligible assignments instead of permanent
- Activation requires justification and (where configured) approval
- Activation logged for audit
- Eligible roles: Owner at subscription level, Key Vault Administrator on prod KVs, anything that grants role-assignment rights

## Onboarding a new team member

1. Add to the appropriate Entra group (`platform-team`, `support-team`, etc.).
2. Group inherits role assignments at the RG level. They do not get personal assignments.
3. PIM eligibility configured for elevated roles they may need.
4. Document the addition in the team's onboarding tracker.

## Offboarding

1. Remove from Entra groups. All RG-level access is gone immediately.
2. Disable any Managed Identities they directly own.
3. Audit recent role assignments they made. Some may need re-issuing under a different owner.

## Quarterly review

Once a quarter:

1. List all role assignments at the subscription and at each RG (`az role assignment list --all`).
2. Cross-check against current team rosters.
3. Remove stale assignments.
4. Document who has Owner, why, and whether it can drop to Contributor.

## References

- Azure RBAC docs: https://learn.microsoft.com/azure/role-based-access-control/overview
- Azure Managed Identity: https://learn.microsoft.com/azure/active-directory/managed-identities-azure-resources/overview
- PIM: https://learn.microsoft.com/azure/active-directory/privileged-identity-management/pim-configure
