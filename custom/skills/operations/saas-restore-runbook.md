---
kind: task-skill
id: saas-restore-runbook
version: 1
title: Restore a BC SaaS environment point-in-time
description: Runbook for Business Central SaaS point-in-time restores. Use when restoring a customer environment from a backup, or planning whether a restore is even possible given limits.
bc-version: [all]
technologies: [al]
countries: [w1]
application-area: [all]
---

# BC SaaS Point-in-Time Restore Runbook

Step-by-step for restoring a Business Central SaaS environment from backup. Designed for support engineers acting under time pressure during incident recovery.

## When to use

- Customer needs to roll back to a known good state (data corruption, bad import, posted-in-error batch)
- Bringing a sandbox in line with a recent production state for testing
- Investigating "is a restore even possible?" before committing to it

## Hard limits (read before promising anything)

| Limit | Value |
|---|---|
| Restores per calendar month | 10 |
| Backup retention window | 28 days |
| Cross-region restore | Not allowed (same Azure region only) |
| Localisation change during restore | Not allowed |
| Sandbox to Production restore | Not allowed |
| Allowed paths | Prod -> Prod, Prod -> Sandbox, Sandbox -> Sandbox |

Production restores **bypass** the sandbox capacity limit intentionally. Sandbox restores **do not** bypass limits, so if the customer is at sandbox cap you will need to temporarily soft-delete an existing sandbox.

## Prerequisites

- The user performing the restore has `D365 BACKUP/RESTORE` permission set
- The customer has a paid BC subscription (trials cannot restore)
- The target environment exists (or we are restoring into a new environment)

## Pre-restore checklist

1. Confirm the desired restore point is within the last 28 days.
2. Confirm Azure region (same as source).
3. Confirm the user account has `D365 BACKUP/RESTORE`.
4. Pause **Job Queues** on the source environment if it is still running.
5. Restrict access on the source environment if recovery is sensitive (use a Security Group, see `security-group-setup`).
6. If reusing the environment name, rename the original environment first by appending `-DONOTUSE`. This avoids a name collision and gives a clear marker for cleanup.
7. Snapshot the list of installed apps (you will reinstall PTEs after restore).

## What gets restored vs cleaned

After a restore, BC's default cleanup:

- **Restored**: all business data, posted documents, master data, setup data, dimensions, journal entries
- **Restored**: AppSource apps, at the **latest hotfix** even if newer than the restore point. This is a Microsoft policy, not configurable.
- **Not restored**: dev-only extensions installed from VS Code (these are not in the backup, you must reinstall manually)
- **Disabled**: Document Exchange, Currency Exchange Rates, VAT Registration validation, Graph Mail, CRM/CDS connections, webhooks
- **Cleared**: OCR passwords, SMTP config, Exchange URLs, Outlook REST accounts

The "Disabled" and "Cleared" categories are the most surprising. Restored environments come up with integrations off so they cannot accidentally fire on stale data.

## Restore steps

1. Open the **Business Central Admin Center**.
2. Select the **source environment** (the one you are restoring from).
3. **Backup & Restore** > **New Restore**.
4. Pick the **restore point** (date and time, within 28 days).
5. Pick the **target environment** (new or existing, name must be available).
6. Confirm and submit.
7. Restore typically takes 30 minutes to 2 hours depending on data volume. The admin center shows progress.

## Post-restore checklist

1. Verify all expected AppSource apps are installed at the right version.
2. Reinstall any PTE (per-tenant extension) apps that were not from AppSource.
3. Re-enable disabled integrations one by one, testing each:
   - Document Exchange
   - Currency Exchange Rate Services
   - VAT Registration validation
   - Graph Mail / SMTP / Outlook
   - CRM / Dataverse / Customer Insights
4. Reconfigure any webhook subscriptions (they were dropped).
5. Re-add API consumers (the keys, not the integration logic).
6. Run **smoke tests**:
   - Sales: create a quote, convert to order, post a shipment, post an invoice
   - Purchase: create a PO, receive, post the invoice
   - Finance: post a general journal, run trial balance
   - Reporting: run one customer-visible report end-to-end
7. Unrestrict access (remove the security group lock if you set one).
8. Notify the customer the environment is back.
9. Delete the `-DONOTUSE` original environment when the customer confirms the restored one is good.

## What to tell the customer

Before:
- Estimated downtime (30 min to a few hours)
- Integrations will come up disabled and need manual reconnection
- Any work done after the restore point is gone

After:
- The smoke tests you ran
- Which integrations need them to re-enter credentials
- The new environment name and any access changes

## Related skills

- `security-group-setup` for restricting access during the restore window
- `al-go-environment-onboarding` if the restored environment also needs CI/CD wired back up

## References

- Microsoft docs on BC SaaS restore: https://learn.microsoft.com/dynamics365/business-central/dev-itpro/administration/tenant-admin-center-backup-restore
