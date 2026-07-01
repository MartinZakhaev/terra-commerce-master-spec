# ADR-0022 — Use Check Constraints and Defer RLS and Partitioning

**Status:** Final  
**Date:** 2026-07-01  
**Decision Owners:** Technical Lead, Database Lead, Security Owner

## Context

The database design left open the representation of evolving states, Row-Level Security, partitioning thresholds, and baseline retention behavior.

## Decision

For MVP:

- Store evolving business states in text or varchar columns with database `CHECK` constraints synchronized with final state-machine values.
- Do not use native PostgreSQL enum types for frequently evolving business states.
- Defer PostgreSQL Row-Level Security. Tenant-aware repositories, composite tenant-safe foreign keys, authorization, and cross-tenant tests remain mandatory.
- Do not partition tables initially.
- Review partitioning when an append-only table approaches 10 million rows, 20 GB, or shows measured vacuum, retention, index, or query-maintenance problems.
- Never automatically delete order, payment, refund, inventory-movement, shipment, audit, or transition history during MVP.
- Operational event, webhook, outbox, job, and generated-file retention is configuration-backed and requires an approved policy before deletion.

## Alternatives Considered

- Native PostgreSQL enums: rejected because state changes require more rigid database migrations.
- RLS in MVP: deferred because application and composite-key controls are already required and RLS adds policy/migration complexity.
- Partitioning from the beginning: rejected because current volume does not justify operational overhead.

## Consequences

Positive: easier state evolution, simpler MVP operations, and measured rather than speculative partitioning.

Negative: application tenant-scoping defects are not independently blocked by RLS, increasing the importance of repository discipline and negative tests.

## Review Trigger

Review after measured growth thresholds, a tenant-isolation incident, compliance requirements, or a formal retention policy.