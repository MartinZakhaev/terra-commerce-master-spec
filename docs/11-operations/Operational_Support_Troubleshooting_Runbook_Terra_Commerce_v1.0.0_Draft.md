# Operational Support and Troubleshooting Runbook — Terra Commerce

**Document ID:** OPS-TC-004  
**Version:** 1.0.0  
**Status:** Draft  
**Owner:** Operations Lead and Support Lead  
**Last Updated:** 2026-07-01  
**Upstream:** Final Architecture, TDD Suite, Authorization Matrix, Threat Model, Observability Specification, and Quality Strategy

## 1. Purpose

Provides standard diagnosis and recovery procedures for tenant onboarding, access, suspended tenants, failed jobs, outbox/Redis backlogs, SSE degradation, provider failures, reports, disk pressure, certificates, and secret rotation.

## 2. Operating Rules

- Use correlation IDs and release/version context.
- Preserve tenant isolation and least privilege.
- Support mode requires explicit tenant, reason, expiry, and audit.
- Do not bypass state machines or edit authoritative data manually without an approved corrective procedure.
- Prefer idempotent replay/retry tools over direct database mutation.
- Escalate security, financial, or data-integrity anomalies immediately.

## 3. Standard Triage

1. Identify affected tenant, user/customer, resource, operation, and time window.
2. Obtain correlation/event/provider IDs.
3. Check platform health, release, deployment, database, Redis, worker, and provider dashboards.
4. Confirm whether issue is isolated, tenant-wide, or platform-wide.
5. Review sanitized logs, audit, transitions, outbox, and job status.
6. Apply the least invasive documented recovery.
7. Validate outcome and record actions.

## 4. Tenant Onboarding

Checklist:

- tenant, owner, plan, store, warehouse, roles, and feature state created.
- owner invitation delivered or recoverable.
- tenant status allows intended operations.
- store/warehouse configuration validates.
- product/user limits known.
- storefront tenant resolution works.

If partial provisioning occurs, use idempotent provisioning recovery or rollback workflow; do not duplicate tenant/owner resources.

## 5. Owner and User Access

- Verify membership and user status.
- Verify role assignment and session revocation events.
- Resend or revoke invitation using authorized APIs.
- Owner access reset requires approved workflow and audit.
- Never reveal password hash, reset token, or session secret.
- Preserve at least one active owner.

## 6. Tenant Suspension/Deactivation

- Confirm requested state and reason.
- Verify sessions and SSE connections terminate.
- Confirm normal mutations are blocked.
- Support access remains time-limited and audited.
- Reactivation requires valid plan/configuration and state transition.

## 7. Failed Jobs and Dead Letters

1. Identify consumer/job, tenant, aggregate, event ID, attempts, and error class.
2. Determine transient, permanent, version-gap, or security failure.
3. Validate current aggregate state before replay.
4. Fix dependency/configuration/root cause.
5. Replay with permission, reason, idempotency, and audit.
6. Confirm no duplicate side effect.

Never replay malformed, tenant-mismatched, or security-quarantined events without Security review.

## 8. Outbox and Redis Backlog

Check:

- oldest pending outbox age.
- rows stuck in publishing.
- Redis connectivity/memory/evictions.
- consumer-group lag and pending entries.
- worker health and concurrency.
- poison/version-gap events.

Recovery:

- restore Redis connectivity.
- reclaim stale claims/entries after lease expiry.
- restart workers gracefully.
- increase concurrency only within approved capacity.
- preserve stable event IDs and consumer receipts.

## 9. SSE Degradation

- Verify gateway readiness, Redis/fan-out, connection count, replay store, proxy buffering, and tenant/session status.
- Confirm connection limits and slow-consumer disconnects.
- Clients may be instructed to refresh APIs; SSE is not authoritative.
- Terminate compromised or unauthorized streams.

## 10. Biteship/Provider Failures

- Check provider health, credentials, rate limits, timeout, and circuit breaker.
- Distinguish confirmed failure from unknown outcome.
- Reconcile by stable provider reference before retrying shipment creation.
- Quarantine conflicting/unknown webhooks.
- Rotate verification credentials under incident procedure if compromise is suspected.

## 11. Checkout, Payment, and Refund Issues

- Use order, payment state, idempotency, reservation, transition, and provider references.
- Never create a second order/payment/refund to compensate before checking prior idempotent outcome.
- Reconcile unknown provider result.
- Validate captured/refunded totals and dual-control status.
- Escalate mismatches or duplicate effects as security/financial incident.

## 12. Inventory and WMS Issues

- Compare physical, reserved, incoming, damaged buckets and movement ledger.
- Check open reservations, receiving, picking/packing tasks, and exceptions.
- Corrections require authorized adjustment with reason and audit.
- Never delete movement or transition history.

## 13. Reports and Downloads

- Check report state, worker error, query limits, object key, expiry, and requester authorization.
- Regenerate derived reports when safe.
- Download requires current membership, permission, ownership, and non-expired link.
- Investigate unusual bulk export as privacy/security event.

## 14. Disk and Resource Pressure

- Identify database, logs, images, temporary reports, Redis persistence, or container layers.
- Stop heavy reports and non-critical jobs if necessary.
- Rotate/compress/delete only approved ephemeral or expired data.
- Never delete PostgreSQL data, active backups, audit, or authoritative files manually.
- Scale or migrate according to capacity triggers.

## 15. Certificates and Secrets

Certificates:

- monitor expiry.
- renew and validate chain/HTTPS/SSE/webhook routes.
- retain rollback certificate/configuration.

Secrets:

- follow rotation playbook.
- update dependent services in controlled order.
- validate authentication and provider connectivity.
- revoke old secret after successful cutover.
- preserve audit without logging values.

## 16. Escalation Rules

Escalate immediately for tenant/customer leakage, financial discrepancy, irreversible data corruption, provider-secret compromise, failed restore, missing audit evidence, repeated OOM/disk criticality, or unknown broad impact.

## 17. Support Evidence

Record ticket/incident, tenant, resource identifiers, correlation IDs, release version, observations, commands/actions, approvals, result, and follow-up tasks. Redact sensitive data.

## 18. Acceptance Criteria

1. Common platform and tenant failures have safe procedures.
2. Retries/replays remain idempotent and state aware.
3. Support cannot bypass tenant/security controls.
4. Financial and data-integrity anomalies escalate.
5. Every action is evidenced and auditable.

## Change Summary

Initial operational support and troubleshooting runbook draft synchronized with finalized technical, security, and observability requirements.
