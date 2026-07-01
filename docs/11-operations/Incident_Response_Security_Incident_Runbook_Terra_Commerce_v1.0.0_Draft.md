# Incident Response and Security Incident Runbook — Terra Commerce

**Document ID:** OPS-TC-003  
**Version:** 1.0.0  
**Status:** Draft  
**Owner:** Security Owner and Operations Lead  
**Last Updated:** 2026-07-01  
**Upstream:** Security Threat Model, Authorization Matrix, Observability Specification, Quality Strategy, and Release Checklist

## 1. Purpose

Defines detection, classification, containment, evidence preservation, eradication, recovery, communication, and post-incident review for operational and security incidents.

## 2. Severity

- SEV-1: tenant/customer data exposure, financial corruption, active credential compromise, total outage, unrecoverable data risk.
- SEV-2: critical journey unavailable, major provider/database/worker failure, imminent resource exhaustion.
- SEV-3: degraded service, bounded backlog, elevated errors, non-critical security control failure.
- SEV-4: warning, anomaly, or planned corrective action.

## 3. Roles

- Incident Commander.
- Security Lead.
- Operations Lead.
- Technical/Service Lead.
- Communications Owner.
- Evidence Custodian.
- Product/Business Owner.

One person may fill multiple roles for MVP, but command and evidence responsibilities must remain explicit.

## 4. Incident Intake

Record:

- detection time/source
- affected service/tenant/data
- current severity
- release/configuration version
- indicators and alerts
- known customer/business impact
- immediate containment actions
- incident channel and owner

## 5. First Response

1. Acknowledge alert and assign commander.
2. Validate incident without destroying evidence.
3. Stop active harm: revoke credentials/sessions, disable route/feature, isolate host/container, enable maintenance mode, or block provider traffic as appropriate.
4. Preserve logs, audit records, metrics, images, configuration fingerprints, and database snapshots where safe.
5. Establish communication cadence.
6. Reassess severity continuously.

## 6. Security-Specific Playbooks

### Cross-Tenant Exposure

- Stop affected operation/route.
- Preserve request, audit, query, and release evidence.
- Identify affected tenants/resources/time window.
- Revoke risky sessions and support access.
- Patch tenant filter/authorization issue.
- Run full cross-tenant regression before restoration.

### Credential or Secret Exposure

- Revoke/rotate immediately.
- Identify dependent services and blast radius.
- Search logs/artifacts/source history for exposure.
- Redeploy with trusted credentials.
- Revoke sessions if authentication material is affected.

### Biteship/Webhook Compromise

- Rotate verification credential/token.
- Quarantine unverified events.
- Reconcile shipment states from trusted provider data.
- Reprocess only verified/idempotent events.

### Financial/Refund Anomaly

- Stop affected commands.
- Preserve payment/refund/idempotency/transition records.
- Prevent duplicate retries.
- Reconcile provider and local amounts.
- Require dual-control review for corrective action.

### Malicious Support Session

- Terminate session.
- Revoke privileged account/session.
- Preserve real/effective actor audit evidence.
- Review every accessed/mutated tenant resource.

## 7. Operational Playbooks

- Database outage/corruption: invoke DR runbook.
- Redis outage: stop unsafe claims, preserve PostgreSQL outbox, degrade SSE/cache/rate-limit according to policy.
- Worker backlog: cap inputs where possible, increase only approved concurrency, inspect poison events.
- Provider outage: circuit-break, queue/retry safely, communicate customer impact.
- Disk/resource pressure: stop heavy reports/log growth, preserve authoritative data, scale or clean only approved ephemeral/expired data.

## 8. Evidence Preservation

- Use UTC timestamps.
- Preserve original logs and checksums.
- Record commands/actions and operators.
- Avoid modifying append-only/audit records.
- Restrict evidence access.
- Retain according to incident/legal policy.

## 9. Communication

Communication states facts, impact, affected scope, mitigations, and next update. Do not expose secrets, exploit details, or unverified conclusions. Tenant/customer notification follows legal and business decision processes.

## 10. Recovery Gates

Before restoration:

- root cause or safe containment understood.
- compromised credentials rotated.
- patched artifacts verified by provenance.
- security and tenant-isolation tests pass.
- data/invariant reconciliation passes.
- monitoring and alerting active.
- rollback available.
- commander approves staged restoration.

## 11. Post-Incident Review

Within the defined review window:

- timeline and impact.
- root cause and contributing factors.
- detection and response effectiveness.
- data/customer impact.
- corrective actions with owner and deadline.
- document/test/runbook changes.
- risk acceptance or architecture decision where necessary.

## 12. Tabletop Exercises

At minimum exercise cross-tenant leak, provider-secret compromise, database credential compromise, support-mode abuse, host loss, and duplicate financial side effect.

## 13. Acceptance Criteria

1. Severity, ownership, and escalation are explicit.
2. Immediate containment does not destroy evidence.
3. Rotation and session revocation are executable.
4. Recovery requires security and data validation.
5. Corrective actions update specifications and tests.

## Change Summary

Initial incident response and security incident runbook draft aligned with finalized security and operational controls.
