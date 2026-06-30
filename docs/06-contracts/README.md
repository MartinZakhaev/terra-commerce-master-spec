# Contracts

This directory contains the authoritative Terra Commerce public API, domain event, SSE, and Biteship webhook contracts.

## Current authoritative contracts

- [`OpenAPI_Contract_Terra_Commerce_v1.0.0_Final.md`](OpenAPI_Contract_Terra_Commerce_v1.0.0_Final.md) — public gateway endpoint inventory, command metadata, errors, idempotency, concurrency, and naming rules.
- [`Domain_Event_Contract_Terra_Commerce_v1.0.0_Final.md`](Domain_Event_Contract_Terra_Commerce_v1.0.0_Final.md) — durable domain-event envelope, catalog, ordering, compatibility, registry, and consumer requirements.
- [`SSE_Contract_Terra_Commerce_v1.0.0_Final.md`](SSE_Contract_Terra_Commerce_v1.0.0_Final.md) — public real-time event projection, audience filtering, replay, limits, and client behavior.
- [`Biteship_Webhook_Contract_Terra_Commerce_v1.0.0_Final.md`](Biteship_Webhook_Contract_Terra_Commerce_v1.0.0_Final.md) — normalized Biteship webhook verification, deduplication, state mapping, atomicity, acknowledgement, and retry behavior.

## Historical drafts

- `OpenAPI_Contract_Terra_Commerce_v1.0.0_Draft.md`
- `Domain_Event_Contract_Terra_Commerce_v1.0.0_Draft.md`
- `SSE_Contract_Terra_Commerce_v1.0.0_Draft.md`
- `Biteship_Webhook_Contract_Terra_Commerce_v1.0.0_Draft.md`

## Reviews

- [`reviews/Contract_Suite_Audit_v1.0.0.md`](reviews/Contract_Suite_Audit_v1.0.0.md) — draft-suite audit with five medium-priority findings.
- [`reviews/Contract_Suite_Final_Verification_v1.0.0.md`](reviews/Contract_Suite_Final_Verification_v1.0.0.md) — PASS; all findings closed.

## Downstream authority

These contracts govern `openapi.yaml`, request/response schemas, event schema files, gateway routing, consumer implementations, SSE projectors, Biteship adapters, contract tests, and integration tests.

## Next recommended work

Create the Authorization Matrix and Security Threat Model, then verify every protected endpoint, event audience, state transition, and tenant-ownership rule against this final contract suite.