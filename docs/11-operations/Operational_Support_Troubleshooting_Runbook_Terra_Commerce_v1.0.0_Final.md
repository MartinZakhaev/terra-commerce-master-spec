# Operational Support and Troubleshooting Runbook — Terra Commerce

**Document ID:** OPS-TC-004  
**Version:** 1.0.0  
**Status:** Final  
**Owner:** Operations Lead and Support Lead  
**Last Updated:** 2026-07-01  
**Upstream:** Final Architecture, TDD Suite, Authorization Matrix, Threat Model, Observability Specification, Quality Strategy  
**Audit:** `../09-infrastructure/reviews/Infrastructure_Operations_Suite_Audit_v1.0.0.md`

## 1. Purpose

Provides safe diagnosis and recovery for tenant onboarding, access, suspensions, failed jobs, outbox/Redis backlogs, SSE, providers, checkout/payment/refund, inventory/WMS, reports, disk pressure, certificates, and secret rotation.

## 2. Operating Rules

- Use correlation, event, provider, resource, tenant, and release identifiers.
- Preserve tenant isolation and least privilege.
- Support mode requires tenant, reason, expiry, and audit.
- Do not bypass state machines.
- Prefer idempotent replay/retry tools.
- Escalate security, financial, or integrity anomalies immediately.
- Ad hoc production SQL mutations are prohibited.

## 3. Standard Triage

1. Identify tenant, actor, resource, operation, and time window.
2. Gather correlation/event/provider IDs.
3. Check platform, release, PostgreSQL, Redis, workers, providers, and alerts.
4. Determine isolated, tenant-wide, or platform-wide scope.
5. Review sanitized logs, audit, transitions, outbox, and jobs.
6. Apply the least invasive documented action.
7. Validate and record the outcome.

## 4. Tenant and Access Support

Verify tenant, owner, plan, store, warehouse, membership, role, invitation, session, and tenant status. Use approved invitation/access-reset APIs. Preserve at least one owner and never expose credentials or reset tokens.

Suspension/deactivation must terminate sessions/SSE and block normal mutations. Reactivation requires a valid state transition and configuration.

## 5. Failed Jobs and Dead Letters

Identify consumer/job, tenant, aggregate, event ID, attempts, and error class. Validate current state, fix root cause, then replay with permission, reason, idempotency, and audit. Security-quarantined or tenant-mismatched events require Security review.

## 6. Outbox and Redis Backlog

Inspect oldest outbox age, publishing claims, Redis health, consumer lag/pending entries, worker health, and poison/version-gap events.

Restore connectivity, reclaim only expired claims/leases, restart gracefully, and raise concurrency only within approved capacity. Preserve stable event IDs and durable consumer receipts.

## 7. SSE

Check gateway readiness, Redis/fan-out, replay store, proxy buffering, connection limits, tenant/session/permission status, and slow consumers. APIs remain authoritative. Terminate unauthorized streams.

## 8. Providers, Checkout, Payment, and Refund

Distinguish confirmed provider failure from unknown outcome. Reconcile using stable provider/idempotency references before retrying.

For payment/refund issues, inspect order/payment/refund state, amounts, idempotency, reservations, transitions, dual control, and provider state. Never create a second operation before resolving the first outcome.

## 9. Inventory and WMS

Compare physical, reserved, incoming, damaged buckets and movement history. Check reservations, receiving, tasks, and exceptions. Corrections require authorized adjustments with reason and audit; movement/transition history is never deleted.

## 10. Reports and Exports

Check report state, worker error, limits, object reference, expiry, requester, tenant, and current permission. Regenerate derived reports when safe. Unusual bulk export is a privacy/security signal.

## 11. Resource, Certificate, and Secret Operations

For resource pressure, stop heavy work, identify the source, and remove only approved ephemeral or expired data. Never manually delete PostgreSQL data, active backups, audit, or authoritative objects.

Certificate renewal verifies chain, HTTPS, SSE, and webhook routes. Secret rotation follows dependency order, overlap, validation, revocation, and rollback without logging values.

## 12. Break-Glass Data Correction

Direct ad hoc SQL is forbidden. A production data correction requires:

- incident/change reference and business justification
- exact reviewed script/query committed or preserved as an artifact
- tenant scope and affected-row preview
- backup/checkpoint before execution
- explicit transaction and rollback plan
- two-person technical review and required business/security approval
- dry run in a representative non-production environment
- execution by a restricted break-glass identity
- audit/log capture without sensitive leakage
- invariant, tenant-isolation, projection, outbox, and provider validation afterward
- post-change review and permanent code/process fix

If these controls cannot be met, use application workflows, migration, or disaster-recovery procedures instead.

## 13. Escalation

Immediate escalation applies to tenant/customer leakage, financial discrepancy, irreversible corruption, provider-secret compromise, failed restore, missing audit evidence, repeated OOM/disk criticality, or unknown broad impact.

## 14. Evidence

Record ticket/incident, tenant/resource IDs, correlations, release, observations, actions/commands, approvals, result, and follow-ups. Redact sensitive data.

## 15. Acceptance Criteria

1. Common failures have state-aware, idempotent procedures.
2. Support cannot bypass tenant/security controls.
3. Ad hoc production mutation is prohibited.
4. Break-glass corrections are reviewed, reversible, audited, and validated.
5. Financial and integrity anomalies escalate.

## Change Summary

Final version closes OPS-M06 by explicitly prohibiting ad hoc production SQL and defining a controlled break-glass data-correction process.
