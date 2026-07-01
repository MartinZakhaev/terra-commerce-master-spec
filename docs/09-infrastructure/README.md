# Infrastructure

This directory contains the authoritative Terra Commerce environment, configuration, observability, capacity, and infrastructure-operations specifications.

## Current authoritative documents

- [`Environment_Configuration_Specification_Terra_Commerce_v1.0.0_Final.md`](Environment_Configuration_Specification_Terra_Commerce_v1.0.0_Final.md)
- [`Observability_Capacity_Specification_Terra_Commerce_v1.0.0_Final.md`](Observability_Capacity_Specification_Terra_Commerce_v1.0.0_Final.md)

## Machine-readable configuration

- [`schemas/configuration.schema.json`](schemas/configuration.schema.json) — JSON Schema Draft 2020-12 entrypoint.
- [`schemas/configuration/`](schemas/configuration/) — 64 runtime settings across gateway, commerce, workers, PostgreSQL, Redis, SSE, security, providers, observability, and shared policy.

Every setting declares type, ownership, source, secret classification, mutability, and restart behavior. Credential-bearing settings also declare rotation policy. Production requires CSRF protection and disallows debug logging.

## Reviews

- [`reviews/Infrastructure_Operations_Suite_Final_Verification_v1.0.0.md`](reviews/Infrastructure_Operations_Suite_Final_Verification_v1.0.0.md) — PASS.
- [`../06-contracts/reviews/Implementation_Foundation_Verification_v1.0.0.md`](../06-contracts/reviews/Implementation_Foundation_Verification_v1.0.0.md) — PASS for the machine-readable implementation foundation.

## Related operational runbooks

See [`../11-operations/`](../11-operations/).

## Downstream authority

These specifications and schemas govern environment templates, startup validation, secret rotation, release manifests, configuration drift checks, and CI validation.