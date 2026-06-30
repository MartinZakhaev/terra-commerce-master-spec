# Technical Design

This directory contains the authoritative implementation-oriented designs for Terra Commerce.

## Current authoritative documents

- [`Go_Gateway_TDD_Terra_Commerce_v1.0.0_Final.md`](Go_Gateway_TDD_Terra_Commerce_v1.0.0_Final.md) — gateway middleware, tenant resolution, authorization, internal trust, idempotency, concurrency, SSE, webhook ingress, observability, and tests.
- [`Medusa_Terra_Commerce_Modules_TDD_v1.0.0_Final.md`](Medusa_Terra_Commerce_Modules_TDD_v1.0.0_Final.md) — Medusa ownership, Terra modules, checkout/payment transaction pattern, inventory, OMS/WMS, shipments, returns, refunds, audit, and outbox.
- [`Worker_Event_Pipeline_TDD_Terra_Commerce_v1.0.0_Final.md`](Worker_Event_Pipeline_TDD_Terra_Commerce_v1.0.0_Final.md) — outbox publisher, Redis Streams, consumer idempotency, ordering, retries, dead letters, notifications, SSE projection, reports, delivery, and recovery.
- [`Infrastructure_Deployment_TDD_Terra_Commerce_v1.0.0_Final.md`](Infrastructure_Deployment_TDD_Terra_Commerce_v1.0.0_Final.md) — container topology, network isolation, storage, resource limits, CI/CD, schema compatibility, migration, backup, restore, monitoring, and runbooks.

## Historical drafts

- `Go_Gateway_TDD_Terra_Commerce_v1.0.0_Draft.md`
- `Medusa_Terra_Commerce_Modules_TDD_v1.0.0_Draft.md`
- `Worker_Event_Pipeline_TDD_Terra_Commerce_v1.0.0_Draft.md`
- `Infrastructure_Deployment_TDD_Terra_Commerce_v1.0.0_Draft.md`

## Reviews

- [`reviews/Technical_Design_Suite_Audit_v1.0.0.md`](reviews/Technical_Design_Suite_Audit_v1.0.0.md) — draft-suite audit with six medium-priority findings.
- [`reviews/Technical_Design_Suite_Final_Verification_v1.0.0.md`](reviews/Technical_Design_Suite_Final_Verification_v1.0.0.md) — PASS; all findings closed.

## Downstream authority

These documents govern application repository scaffolding, package/module boundaries, implementation patterns, infrastructure code, CI/CD, database migrations, contract tests, load tests, and operational runbooks.

## Next recommended work

Create the Quality and Test Strategy suite, followed by operational runbooks and release-readiness checklists, before application repository scaffolding and implementation begin.