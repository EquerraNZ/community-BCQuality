---
kind: task-skill
id: security-group-setup
version: 1
title: Restrict a BC environment with an Entra security group
description: Restrict access to a Business Central environment using a Microsoft Entra security group. Use when locking down a production environment, setting up a UAT environment for a controlled audience, or restricting a sandbox to a specific team.
bc-version: [all]
technologies: [al]
countries: [w1]
application-area: [all]
---

# BC Security Group Setup

How to restrict access to a Business Central environment using a Microsoft Entra (Azure AD) security group. Once bound, only users who hold a BC licence **and** are members of the group can sign in.

## When to use

- Locking down a Production environment to a specific user set
- Restricting a UAT environment to the test cohort
- Creating a sensitive sandbox (e.g. for payroll data, customer migration) that should not be open to all internal users
- Reviewing or rotating membership of an existing restricted environment

## Pre-requisites

- Access to the **Business Central Admin Center** for the tenant
- An Entra (Azure AD) role that lets you create or manage Security groups (Groups Administrator, or coordinate with the customer)
- The list of users who should have access

## Step-by-step

### 1. Create the Entra security group

1. Open the **Azure portal** > **Microsoft Entra ID** > **Groups** > **New group**.
2. **Group type**: Security.
3. **Name**: descriptive and consistent. Suggested pattern: `<customer>-bc-<env>-access`. Example: `acme-bc-uat-access`.
4. Add **Owners** (at least two, never one).
5. Add initial **Members**.
6. Save.

Use **separate groups per environment**. Production, UAT, and specialised sandboxes each get their own group. Do not reuse a Production group for UAT.

### 2. Bind the group to the BC environment

1. Open the **Business Central Admin Center** for the tenant.
2. Pick the environment.
3. **Security Group** > **Define**.
4. Search for the Entra group by name.
5. **Save**.

Once saved:

- Only users with a **BC licence** AND **group membership** can sign in.
- Delegated admins (partners) always retain access regardless of the group, by design.

### 3. Verify

- Sign in as a user who is in the group. Should succeed.
- Sign in as a user with a BC licence who is **not** in the group. Should be denied with a clear message.
- Confirm delegated admin access still works.

## Operational guidance

- **Prefer group membership over direct user assignment.** Adding users individually to BC and skipping the group means you lose the audit trail and central control.
- **Review group membership quarterly.** Calendar it. Names rot fast; ex-employees with a stale licence are a real risk.
- **Use Conditional Access for sensitive groups.** Entra Conditional Access can layer MFA or location restrictions on top of group membership.
- **Document the group's owner contact.** When an access ticket lands, support needs to know who can add users.

## Common confusions

| Question | Answer |
|---|---|
| Does the group control licence assignment? | No. The group controls who can sign in to the bound environment. Licences are managed separately. |
| Can a delegated admin be locked out? | No. Delegated admins always retain access. This is by Microsoft design and not configurable. |
| Can I bind multiple groups? | Yes, but keep it simple. If you need "Group A OR Group B", create a parent dynamic group. |
| Will removing the binding remove user access? | Yes. The environment goes back to "all licensed users can sign in". Re-bind quickly if that was not the intent. |

## Related skills

- `saas-restore-runbook` for the typical case of restricting access during a restore
- `rbac-and-access` for the broader Azure access model

## References

- Microsoft docs on BC security groups: https://learn.microsoft.com/dynamics365/business-central/dev-itpro/administration/tenant-admin-center-environments-security-groups
