---
kind: task-skill
id: bc-integrations
version: 1
title: Integrate BC with external systems
description: Patterns and anti-patterns for integrating Business Central with external systems.
bc-version: [all]
technologies: [al]
countries: [w1]
application-area: [all]
---

# BC Integrations

## When to use

Designing or reviewing any integration between Business Central and an external system (Shopify, 3PL, WMS, custom services, partner platforms).

## Architectural choices

### Direct API vs middleware

Use Azure Integration Services (Logic Apps, Service Bus, APIM) as the integration plane. Do not call third-party APIs directly from AL. Reasons:

- Retry, dead-letter, and observability sit in the integration plane.
- BC stays free of credential management for external systems.
- Schema evolution on the third-party side does not break BC.

Exceptions: simple one-shot lookups (currency conversion, address validation) can use `HttpClient` from AL directly, with timeouts and explicit error handling.

### Inbound to BC

External system to BC: prefer the API publisher pattern with custom API pages. Use OData for typed access, SOAP only when the consumer cannot do OData.

### Outbound from BC

BC to external: publish business events. Never poll BC from outside if events are available.

For a subscriber outside BC, declare an `[ExternalBusinessEvent]` (not the in-process `[BusinessEvent]`) and fire it from a thin subscriber on the real event (release, post). Delivery is asynchronous and post-commit, so it is safe to fire from a posting or release path and nothing is delivered if the transaction rolls back. The external subscriber registers directly by POSTing to `api/microsoft/runtime/v1.0/externaleventsubscriptions` with a `notificationUrl` and a `clientState` (the shared secret echoed on every notification); it needs the `Ext. Events - Subscr` permission set. BC delivers by HTTP webhook to that URL. There is no direct Service Bus or Event Grid delivery for BC, so to land events on a queue, point `notificationUrl` at a thin Function that forwards to it. See `al-modern-integration-patterns` for the citable rules and `specs/contracts/integration-contract.md` for a worked example.

## Common patterns

### Idempotency

Every inbound write needs an idempotency key. Project convention: external system passes a correlation ID, BC stores it on the record, repeated calls with the same ID are no-ops.

### Sync vs replicate

- Sync: BC is the source of truth, external mirrors.
- Replicate: external is the source of truth, BC mirrors.
- Hybrid: each field has a defined owner. Document it.

Pick one and write it down. Implicit sync direction causes the worst integration bugs.

### Error escalation

- Transient errors: retry with backoff in the integration plane.
- Permanent errors: dead-letter, notify the support inbox, log to telemetry.
- Business-rule rejections: surface to the user via the relevant role centre cue.

## Anti-patterns

- Polling BC every 30 seconds for changes. Use events.
- Storing external credentials in BC isolated storage. Use Key Vault via the integration plane.
- One giant payload containing all entities. Split per business object.
- Direct DB-to-DB sync that bypasses BC business logic.
