# Infrastructure

This directory contains the authoritative Terra Commerce environment, configuration, observability, capacity, and infrastructure-operations review documents.

## Current authoritative documents

- [`Environment_Configuration_Specification_Terra_Commerce_v1.0.0_Final.md`](Environment_Configuration_Specification_Terra_Commerce_v1.0.0_Final.md) — environment classes, configuration registry, source of truth, mutability, validation, secrets, rotation, promotion, drift, and recovery.
- [`Observability_Capacity_Specification_Terra_Commerce_v1.0.0_Final.md`](Observability_Capacity_Specification_Terra_Commerce_v1.0.0_Final.md) — logs, metrics, dashboards, alert design, SLO/burn-rate guidance, capacity forecasts, storage gates, and scaling triggers.

## Historical drafts

- `Environment_Configuration_Specification_Terra_Commerce_v1.0.0_Draft.md`
- `Observability_Capacity_Specification_Terra_Commerce_v1.0.0_Draft.md`

## Reviews

- [`reviews/Infrastructure_Operations_Suite_Audit_v1.0.0.md`](reviews/Infrastructure_Operations_Suite_Audit_v1.0.0.md) — draft-suite audit with six medium-priority findings.
- [`reviews/Infrastructure_Operations_Suite_Final_Verification_v1.0.0.md`](reviews/Infrastructure_Operations_Suite_Final_Verification_v1.0.0.md) — PASS; all findings closed.

## Related operational runbooks

See [`../11-operations/`](../11-operations/) for deployment, recovery, incident response, and support procedures.

## Downstream authority

These specifications govern environment templates, configuration schemas, secret rotation, telemetry implementation, dashboards, paging alerts, capacity reviews, and scaling decisions.