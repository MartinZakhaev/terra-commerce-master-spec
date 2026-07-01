# Backup, Restore, and Disaster Recovery Runbook — Terra Commerce

**Document ID:** OPS-TC-002  
**Version:** 1.0.0  
**Status:** Draft  
**Owner:** Operations and DevOps Lead  
**Last Updated:** 2026-07-01  
**Upstream:** Final Architecture, Database Design, Infrastructure TDD, Threat Model, and Release Checklist

## 1. Purpose

Defines backup operation, verification, restore, disaster declaration, recovery validation, and service restoration for PostgreSQL, configuration references, object storage, and event-processing state.

## 2. Recovery Principles

- PostgreSQL is authoritative business truth.
- Backups are encrypted and stored off-host.
- Redis is not the sole recovery source.
- Restore tests are mandatory.
- Recovery preserves tenant isolation, audit history, and release/schema compatibility.
- RPO and RTO targets are owned by operations policy and measured during drills.

## 3. Backup Scope

- PostgreSQL database including audit, transitions, outbox, idempotency, and provider events.
- Migration version and release manifest.
- Non-secret environment templates and approved configuration references.
- Object-storage inventory/reference metadata.
- Optional Redis persistence for operational acceleration, never as sole truth.

## 4. Scheduled Backup Procedure

1. Verify backup destination availability and encryption.
2. Run logical or physical backup according to approved method.
3. Capture database/version/release metadata.
4. Generate checksum and verify readable archive.
5. Transfer off-host.
6. Apply retention policy.
7. Record success/failure and duration.
8. Alert on failure or stale backup.

## 5. Restore Test Procedure

1. Select a verified backup point.
2. Provision isolated restore environment.
3. Restore PostgreSQL.
4. Apply only compatible migrations.
5. Validate constraints and schema version.
6. Validate representative tenants, users, products, orders, payments, inventory, shipments, returns/refunds, audit, transitions, idempotency, and outbox.
7. Verify no cross-tenant reference corruption.
8. Reconcile object references.
9. Test application startup and critical flows.
10. Record achieved RPO/RTO and findings.

## 6. Disaster Declaration

Examples:

- database volume loss/corruption.
- unrecoverable migration damage.
- host loss.
- ransomware or credential compromise.
- widespread data-integrity failure requiring point-in-time recovery.

Incident Commander authorizes recovery point and service restoration.

## 7. Production Recovery Procedure

1. Declare incident and stop mutations.
2. Preserve evidence and current state where safe.
3. Isolate compromised or failed infrastructure.
4. Select restore point based on integrity and RPO.
5. Provision clean target.
6. Restore database and required configuration references.
7. Verify schema compatibility.
8. Reconcile outbox and Redis state.
9. Reconnect object storage/provider integrations carefully.
10. Run tenant-isolation and critical-domain validation.
11. Start services in controlled order: data, commerce, worker paused or limited, gateway, frontends.
12. Reconcile uncertain payment/shipment/provider operations.
13. Restore traffic gradually.
14. Communicate recovery status and known data window.

## 8. Outbox and Redis Reconciliation

- Pending/unpublished outbox events remain publishable.
- Published events may redeliver; consumers deduplicate.
- Redis Streams may be rebuilt or resumed according to available persistence.
- Never mark business work complete based solely on Redis state.
- Reconcile payment and shipment operations with external providers before retrying uncertain commands.

## 9. Object Storage Recovery

- Validate bucket availability, private access, lifecycle rules, and tenant key prefixes.
- Restore or re-link missing reports/labels/media according to provider capability.
- Regenerate derived reports/labels when safe.
- Do not expose objects publicly during recovery.

## 10. Security Recovery

If compromise is suspected:

- rotate database, Redis, provider, storage, email, deployment, and internal trust credentials.
- revoke sessions/tokens.
- verify backup integrity before restore.
- preserve evidence.
- require clean images and trusted provenance.

## 11. Validation Gates

Recovery cannot complete until:

- tenant isolation tests pass.
- financial and inventory invariants pass.
- schema compatibility passes.
- audit/transition history is readable.
- outbox/worker state is understood.
- critical customer and staff journeys pass.
- monitoring and backups are re-enabled.

## 12. Failure During Restore

Stop and reassess when checksum fails, schema is incompatible, tenant references are corrupt, backup contains suspected compromise, or critical validation fails. Do not overwrite the only verified backup copy.

## 13. Evidence

Record restore point, backup checksum, release/schema versions, operators, timestamps, RPO/RTO, validation output, data-loss window, reconciliations, and approvals.

## 14. Acceptance Criteria

1. Backup scope includes all authoritative business and reliability data.
2. Restore is tested in isolation.
3. Redis/outbox recovery respects at-least-once semantics.
4. Security compromise includes rotation and evidence preservation.
5. Traffic resumes only after domain and tenant validation.

## Change Summary

Initial backup, restore, and disaster recovery runbook draft synchronized with finalized data, infrastructure, security, and quality requirements.
