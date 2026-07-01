# Security

This directory contains the authoritative Terra Commerce authorization, threat-model, and machine-readable access-control artifacts.

## Current authoritative documents

- [`Authorization_Matrix_Terra_Commerce_v1.0.0_Final.md`](Authorization_Matrix_Terra_Commerce_v1.0.0_Final.md)
- [`Security_Threat_Model_Terra_Commerce_v1.0.0_Final.md`](Security_Threat_Model_Terra_Commerce_v1.0.0_Final.md)

## Machine-readable registry

- [`operation-permission-registry.yaml`](operation-permission-registry.yaml) — deny-by-default registry index.
- [`operation-permission-rules/`](operation-permission-rules/) — 134 operation rules matching the OpenAPI operation set exactly.

The registry merges each rule with method, path, audience, ownership, idempotency, version, audit, and event metadata from the corresponding OpenAPI operation.

## Reviews

- [`reviews/Authorization_Threat_Model_Final_Verification_v1.0.0.md`](reviews/Authorization_Threat_Model_Final_Verification_v1.0.0.md) — PASS.
- [`../06-contracts/reviews/Implementation_Foundation_Verification_v1.0.0.md`](../06-contracts/reviews/Implementation_Foundation_Verification_v1.0.0.md) — PASS for the machine-readable implementation foundation.

## Downstream authority

These artifacts govern authorization middleware, operation registration, resource ownership, dual-control policies, security tests, and CI equality checks against `openapi.yaml`.