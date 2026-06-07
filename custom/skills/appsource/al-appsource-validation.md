---
kind: task-skill
id: al-appsource-validation
version: 1
title: Validate a BC extension for AppSource submission
description: Validate a Business Central extension for AppSource submission against the marketplace checklist.
bc-version: [all]
technologies: [al]
countries: [w1]
application-area: [all]
---

# AppSource Validation

## When to use

Before submitting any extension to Microsoft AppSource, or before publishing a new version of an already-listed app.

## Checklist

### app.json

- `id` is a stable GUID. Never reuse across apps.
- `name`, `publisher`, `version` match the Partner Center listing exactly.
- `application` and `platform` set to current minimum supported versions.
- `dependencies` include the System Application and Base Application.
- `idRanges` match the project's assigned ID range for the publisher.
- `target` is `Cloud` for AppSource apps.
- `runtime` matches the BC runtime version targeted.
- `showMyCode` is `true` only if intentional (most apps are `false`).

### Signing

- `.app` file is signed via Azure Key Vault using the project's AL-Go GitHub Actions pipeline.
- `NavSip.dll` is installed on the build runner for local signature verification.

### Code

- No `Confirm` dialogs in event subscribers.
- No upgrade codeunits that change without a corresponding `previousVersionTag`.
- All user-facing strings use `Label` declarations with translations available.
- Telemetry tagged with the publisher tag.

### Tests

- Test app exists and runs.
- Coverage report attached to the submission.
- Permission set published and used by tests.

### Partner Center listing

- Search summary 100 characters or under.
- Description leads with the value proposition, not feature list.
- Logo SVG meets size requirements.
- Demo video (if provided) under 90 seconds.
- Privacy policy URL live and reachable.
- Terms and Conditions URL live and reachable.

## Project-specific

- Use the project T&Cs template. Do not hand-roll per app.
- Privacy policy aligned with NZ and Australian privacy law.
- Support email points to the project support inbox, not a personal address.
