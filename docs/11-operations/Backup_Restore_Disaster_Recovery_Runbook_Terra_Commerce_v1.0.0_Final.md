# Backup, Restore, and Disaster Recovery Runbook — Terra Commerce

**Document ID:** OPS-TC-002  
**Version:** 1.0.0  
**Status:** Final  
**Owner:** Operations and DevOps Lead  
**Last Updated:** 2026-07-01  
**Upstream:** Final Architecture, Database Design, Infrastructure TDD, Threat Model, Release Checklist  
**Audit:** `../09-infrastructure/reviews/Infrastructure_Operations_Suite_Audit_v1.0.0.md`

## 1. Purpose

Defines backup execution, verification, restore, disaster declaration, split-brain prevention, provider reconciliation, validation, and controlled service restoration.

## 2. Principles

- PostgreSQL is authoritative.
- Backups are encrypted, off-host, integrity checked, and restore tested.
- Redis is never the only recovery source.
- Recovery preserves tenant isolation, audit history, and schema compatibility.
- Old and restored environments must never accept production traffic simultaneously.

## 3. Backup Scope

- PostgreSQL including audit, transitions, outbox, idempotency, webhook/provider events.
- Migration and release metadata.
- Approved configuration references and non-secret templates.
- Object-storage inventory/reference metadata.
- Optional Redis persistence for acceleration only.

## 4. Scheduled Backup

1. Verify encrypted destination.
2. Run approved logical/physical backup.
3. Capture database, schema, and release metadata.
4. Generate checksum and validate archive readability.
5. Transfer off-host.
6. Apply retention.
7. Record duration/status.
8. Alert on failure or staleness.

## 5. Restore Test

1. Provision isolated target.
2. Restore selected verified backup.
3. Apply only compatible migrations.
4. Validate constraints and schema version.
5. Validate tenants, users, products, orders, payments, inventory, shipments, returns/refunds, audit, transitions, idempotency, and outbox.
6. Run cross-tenant integrity checks.
7. Reconcile object references.
8. Start application stack and run critical flows.
9. Record achieved RPO/RTO and findings.

## 6. Disaster Declaration

Applies to volume loss/corruption, destructive migration failure, host loss, ransomware/credential compromise, or broad integrity failure requiring point-in-time recovery.

Incident Commander selects restore point and authorizes restoration.

## 7. Split-Brain Prevention

Before restored services start:

- disable public routing to the old environment.
- stop old gateway, workers, schedulers, and provider callbacks where possible.
- revoke or rotate old service credentials when compromise or ambiguity exists.
- confirm only one environment owns scheduler locks, stream consumers, webhook processing, and mutation traffic.
- use a recovery-generation marker or equivalent fencing token so stale workers cannot resume writes.

Traffic must not return until fencing is verified.

## 8. Provider Reconciliation Freeze

During recovery, block automatic retry of uncertain payment, refund, and shipment operations until local idempotency records and provider state are reconciled.

Classify each uncertain operation as confirmed success, confirmed failure, pending/unknown, or safe to retry. Only idempotent, verified-safe retries are released.

## 9. Production Recovery

1. Declare incident and stop mutations.
2. Preserve evidence/current state.
3. Fence old environment.
4. Select verified restore point.
5. Provision clean target.
6. Restore PostgreSQL and approved configuration references.
7. Verify schema/service compatibility.
8. Reconcile outbox, Redis, object references, and uncertain provider operations.
9. Start data services, commerce, limited workers, gateway, then frontends.
10. Run tenant-isolation, financial, inventory, shipment, audit, and critical-flow validation.
11. Restore traffic gradually.
12. Communicate recovery status and possible data-loss window.

## 10. Outbox and Redis

Pending outbox events remain publishable. Published events may redeliver and consumers deduplicate. Redis Streams may be rebuilt/resumed. Never infer completed business work solely from Redis state.

## 11. Security Recovery

Rotate database, Redis, provider, storage, email, deployment, and internal-trust credentials as required. Revoke sessions/tokens, verify backup integrity, use trusted images, and preserve evidence.

## 12. Recovery Gates

- old environment fenced.
- tenant isolation passes.
- financial/inventory invariants pass.
- schema compatibility passes.
- audit/transition history readable.
- outbox/worker/provider uncertainty reconciled.
- monitoring and backups active.

## 13. Evidence

Record restore point/checksum, operators, times, schema/release versions, fencing actions, RPO/RTO, validation output, provider reconciliation, data-loss window, and approvals.

## 14. Acceptance Criteria

1. Backups cover authoritative business/reliability data.
2. Restore is tested in isolation.
3. Split-brain is prevented with explicit fencing.
4. Payment/shipment uncertainty is frozen and reconciled.
5. Traffic resumes only after tenant and domain validation.

## Change Summary

Final version closes OPS-M05 by adding split-brain fencing and a provider-reconciliation freeze for uncertain payment, refund, and shipment operations.
