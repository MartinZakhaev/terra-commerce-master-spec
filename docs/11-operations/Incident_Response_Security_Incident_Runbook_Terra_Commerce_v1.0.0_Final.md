# Incident Response and Security Incident Runbook — Terra Commerce

**Document ID:** OPS-TC-003  
**Version:** 1.0.0  
**Status:** Final  
**Owner:** Security Owner and Operations Lead  
**Last Updated:** 2026-07-01  
**Upstream:** Security Threat Model, Authorization Matrix, Observability Specification, Quality Strategy, Release Checklist  
**Audit:** `../09-infrastructure/reviews/Infrastructure_Operations_Suite_Audit_v1.0.0.md`

## 1. Purpose

Defines detection, classification, containment, evidence preservation, eradication, recovery, communication, credential rotation, and post-incident review.

## 2. Severity

- **SEV-1:** tenant/customer data exposure, financial corruption, active credential compromise, total outage, unrecoverable data risk.
- **SEV-2:** critical journey unavailable, major provider/database/worker failure, imminent exhaustion.
- **SEV-3:** degraded service, bounded backlog, elevated errors, non-critical control failure.
- **SEV-4:** warning or anomaly requiring planned action.

## 3. Roles

Incident Commander, Security Lead, Operations Lead, Service Lead, Communications Owner, Evidence Custodian, and Product/Business Owner. One person may hold multiple roles for MVP, but accountability remains explicit.

## 4. Intake

Record detection time/source, affected tenant/service/data, severity, release/configuration version, indicators, business impact, containment actions, incident channel, and owner.

## 5. First Response

1. Assign Incident Commander.
2. Validate without destroying evidence.
3. Stop active harm using route disablement, feature flag, maintenance mode, credential/session revocation, or infrastructure isolation.
4. Preserve logs, audit, metrics, images, configuration fingerprints, and database snapshots where safe.
5. Establish communication cadence.
6. Reassess severity continuously.

## 6. Security Playbooks

### Cross-Tenant Exposure

Stop affected operations, preserve evidence, determine affected tenants/time window, revoke risky access, fix tenant/authorization control, and run full isolation regression before restoration.

### Credential or Secret Exposure

Revoke/rotate immediately, assess blast radius, search source/logs/artifacts, redeploy trusted configuration, and revoke affected sessions.

### Biteship/Webhook Compromise

Rotate verification material, quarantine unverified events, reconcile shipment states from trusted provider data, and replay only verified idempotent events.

### Financial Anomaly

Stop affected commands, preserve payment/refund/idempotency/transition records, prevent blind retries, reconcile provider/local amounts, and require dual control for correction.

### Support-Mode Abuse

Terminate support session, revoke privileged identity/session, preserve real/effective actor evidence, and review every accessed/mutated resource.

## 7. Operational Playbooks

- Database outage/corruption: invoke DR runbook.
- Redis outage: preserve PostgreSQL outbox and stop unsafe claims.
- Worker backlog: inspect poison/version-gap events and use bounded approved concurrency.
- Provider outage: circuit-break and retry safely.
- Resource pressure: stop heavy work and remove only approved ephemeral/expired data.

## 8. Evidence Preservation

Use UTC, preserve originals/checksums, record every operator command/action, restrict evidence access, avoid mutation of audit records, and follow retention/legal policy.

## 9. Communication

Communicate verified facts, impact, scope, mitigation, and next update. Do not expose secrets, exploit details, or speculation. Customer/tenant notification follows legal and business decision procedures.

## 10. Recovery Gates

- Safe containment or root cause understood.
- Compromised credentials rotated.
- Trusted patched artifacts verified.
- Tenant/security tests pass.
- Data and provider reconciliation pass.
- Monitoring active.
- Rollback/recovery path available.
- Incident Commander approves staged restoration.

## 11. Post-Incident Review

Document timeline, impact, root cause, contributing factors, detection/response effectiveness, customer/data impact, corrective actions with owner/deadline, and required changes to specifications, tests, alerts, or runbooks.

## 12. Exercises

Exercise cross-tenant leak, provider-secret compromise, database credential compromise, support abuse, host loss, and duplicate financial effects.

## 13. Acceptance Criteria

1. Severity and ownership are explicit.
2. Containment preserves evidence.
3. Rotation and session revocation are executable.
4. Recovery requires security/data validation.
5. Corrective actions update the controlled baseline.

## Change Summary

Final incident response runbook aligned with the finalized threat model, observability severity model, recovery controls, and release gates.
