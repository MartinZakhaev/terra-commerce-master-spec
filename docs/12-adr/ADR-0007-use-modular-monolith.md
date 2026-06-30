# ADR-0007 — Use a Modular Monolith for the MVP

**Status:** Final  
**Date:** 2026-06-30  
**Decision Owners:** Product Owner, Technical Lead  
**Related:** PRD v1.0.0, FSD v1.0.1

## Context

Terra Commerce must deliver superadmin, tenant administration, commerce, OMS, WMS, delivery, events, and real-time updates on an initial 2 vCPU and 4 GB RAM server. Premature microservices would add deployment, networking, observability, consistency, and operational overhead.

## Decision

Use a modular-monolith architecture for MVP.

The deployment may contain separate processes for the Go gateway, Medusa application, worker, PostgreSQL, Redis, and reverse proxy, but business-domain decomposition remains logical rather than a set of independently deployed microservices.

Required logical modules include:

- Platform and tenant management.
- Catalog.
- Customer and cart.
- Orders and payments.
- Inventory.
- WMS fulfillment.
- Delivery integration.
- Notifications and reporting.
- Audit and authorization.

Module boundaries must be documented and interactions must use explicit interfaces, workflows, or contracts.

## Alternatives Considered

### Microservices from the beginning
Rejected because the operational and resource cost is not justified by MVP scale.

### Single unstructured application
Rejected because it would create tight coupling and make future extraction difficult.

### Serverless decomposition
Rejected because it conflicts with the selected VPS deployment model and increases distributed-operation complexity.

## Consequences

### Positive

- Lower deployment and operational complexity.
- Better fit for constrained infrastructure.
- Easier transactional consistency.
- Faster MVP delivery.

### Negative

- Modules share deployment and may affect one another.
- Independent scaling is limited.
- Poor boundary discipline could create a distributed monolith later.

## Constraints

- Modules must not access another module's internals without an approved interface.
- Cross-module events and workflows must be explicit.
- Heavy jobs run outside synchronous request paths.
- Domain ownership must be reflected in architecture and technical design.
- New independently deployed services require a new ADR.

## Extraction Criteria

A module may become an independent service when at least one is demonstrated:

- Independent scaling requirement.
- Security or compliance isolation.
- Deployment cadence conflict.
- Operational reliability boundary.
- Resource contention supported by measurements.

## Review Trigger

Review after load testing, significant tenant growth, provider isolation needs, or repeated incidents caused by shared deployment boundaries.
