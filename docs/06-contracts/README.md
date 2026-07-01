# Contracts

This directory contains the authoritative Terra Commerce public API, domain-event, SSE, webhook, and machine-readable implementation contracts.

## Current authoritative contracts

- [`OpenAPI_Contract_Terra_Commerce_v1.0.0_Final.md`](OpenAPI_Contract_Terra_Commerce_v1.0.0_Final.md)
- [`Domain_Event_Contract_Terra_Commerce_v1.0.0_Final.md`](Domain_Event_Contract_Terra_Commerce_v1.0.0_Final.md)
- [`SSE_Contract_Terra_Commerce_v1.0.0_Final.md`](SSE_Contract_Terra_Commerce_v1.0.0_Final.md)
- [`Biteship_Webhook_Contract_Terra_Commerce_v1.0.0_Final.md`](Biteship_Webhook_Contract_Terra_Commerce_v1.0.0_Final.md)

## Machine-readable artifacts

- [`openapi.yaml`](openapi.yaml) — OpenAPI 3.1 entrypoint covering 134 operations.
- `openapi-components.json`, `openapi-paths.json`, and domain path files — reusable schemas and operation metadata.
- [`event-registry.yaml`](event-registry.yaml) — 93 events grouped by producer, consumers, aggregate, scope, sensitivity, and schema.
- [`schemas/events/`](schemas/events/) — JSON Schema Draft 2020-12 envelope and event-family data schemas.

## Reviews

- [`reviews/Contract_Suite_Final_Verification_v1.0.0.md`](reviews/Contract_Suite_Final_Verification_v1.0.0.md) — PASS.
- [`reviews/Implementation_Foundation_Verification_v1.0.0.md`](reviews/Implementation_Foundation_Verification_v1.0.0.md) — PASS; machine-readable foundations approved for scaffolding.

## Related registries

- [`../07-security/operation-permission-registry.yaml`](../07-security/operation-permission-registry.yaml)
- [`../09-infrastructure/schemas/configuration.schema.json`](../09-infrastructure/schemas/configuration.schema.json)

## Downstream authority

These artifacts govern generated clients and server interfaces, gateway routing, request validation, event producers and consumers, authorization middleware, contract tests, and CI consistency checks.