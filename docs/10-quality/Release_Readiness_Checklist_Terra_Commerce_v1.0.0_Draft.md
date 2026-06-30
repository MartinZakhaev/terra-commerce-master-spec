# Release Readiness Checklist — Terra Commerce

**Document ID:** QA-TC-006  
**Version:** 1.0.0  
**Status:** Draft  
**Owner:** Product Owner, Technical Lead, QA Lead, Security Owner, and Operations Lead  
**Last Updated:** 2026-06-30  
**Upstream:** All Final Terra Commerce baseline and quality documents

## 1. Purpose

This checklist is the final go/no-go control for a Terra Commerce release. Every release must identify the exact specification versions, submodule commits, image digests, migration version, environment, test evidence, known risks, approvals, and rollback path.

## 2. Release Identity

- [ ] Release ID and version assigned.
- [ ] Target environment identified.
- [ ] Master-spec commit recorded.
- [ ] Every application submodule pinned to an exact approved commit.
- [ ] Container image digests recorded.
- [ ] Database migration version recorded.
- [ ] Configuration/schema compatibility range recorded for gateway, commerce, and worker.
- [ ] Release notes and change summary completed.

## 3. Documentation and Traceability

- [ ] PRD/FSD/architecture/state/data/contracts/security/TDD dependencies remain consistent.
- [ ] Requirements Traceability Matrix updated.
- [ ] Every changed mandatory requirement has implementation and test evidence.
- [ ] Every changed OpenAPI operation has permission mapping and contract tests.
- [ ] Every changed event/SSE/webhook schema has compatibility and consumer tests.
- [ ] ADR created for any new or changed architecture decision.
- [ ] No unresolved document-audit Critical/High finding.

## 4. Code and Review

- [ ] Required pull requests are reviewed and merged.
- [ ] Branch protection and required checks passed.
- [ ] No unreviewed direct production change.
- [ ] Formatting, linting, and static analysis pass.
- [ ] Unit and component tests pass.
- [ ] Code ownership and sensitive-area review requirements satisfied.
- [ ] Generated artifacts match committed sources.

## 5. Database and Migration

- [ ] Migration reviewed for tenant impact, lock duration, backfill, and compatibility.
- [ ] Expand-and-contract used for destructive changes.
- [ ] Migration tested against production-like data volume.
- [ ] Service min/max schema compatibility declared and validated.
- [ ] Dedicated migration job prepared.
- [ ] Pre-deployment backup is current and integrity checked.
- [ ] Rollback or forward-fix procedure documented.
- [ ] Representative restore has passed within the required review period.

## 6. Functional and Integration Quality

- [ ] Critical E2E journeys pass.
- [ ] Platform, tenant, catalog, storefront, checkout, OMS, WMS, shipment, return, refund, event, SSE, report, and audit regression suites pass as applicable.
- [ ] No unresolved Critical defect.
- [ ] No unresolved High defect unless formally accepted with mitigation, owner, and expiry.
- [ ] Provider simulators/sandboxes cover success, timeout, duplicate, and conflict behavior.
- [ ] Idempotency and concurrency tests pass for changed critical operations.

## 7. Security and Privacy

- [ ] OperationId-to-permission registry validation passes.
- [ ] Cross-tenant and cross-customer tests pass fully.
- [ ] State/version/ownership negative tests pass.
- [ ] Biteship webhook verification and replay tests pass.
- [ ] SSE audience/replay/revocation tests pass.
- [ ] Dependency, container, and secret scans pass or findings are formally accepted.
- [ ] No production secret exists in source, image, logs, or test artifact.
- [ ] Dual-control workflows work for configured high-impact actions.
- [ ] Report/export permissions, expiry, and audit tests pass.
- [ ] Critical/High threat-model controls have current evidence.
- [ ] Privileged MFA is enabled where required before production.

## 8. Performance and Capacity

- [ ] Target-host expected-load test passes.
- [ ] API, dashboard, SSE, and job-start targets pass.
- [ ] No OOM, crash loop, or sustained swap.
- [ ] At least 20% steady-state memory headroom at expected peak.
- [ ] Database and Redis remain within safe limits.
- [ ] Worker backlog recovery demonstrated.
- [ ] No unacceptable performance regression against prior baseline.
- [ ] Capacity exceptions documented and approved.

## 9. Infrastructure and Configuration

- [ ] Only approved public ports/routes are exposed.
- [ ] PostgreSQL and Redis remain private.
- [ ] TLS certificate is valid and expiry monitored.
- [ ] Service credentials are least privilege and environment specific.
- [ ] Required environment variables and secrets are present.
- [ ] Startup configuration validation passes.
- [ ] Object storage, email, Biteship, and payment dependencies are configured for target environment.
- [ ] Resource limits and restart policies are reviewed.
- [ ] Log rotation, metrics, dashboards, and alerts are active.

## 10. Operations and Recovery

- [ ] Deployment and rollback runbook reviewed.
- [ ] Migration-failure procedure reviewed.
- [ ] PostgreSQL backup/restore procedure current.
- [ ] Redis/outbox backlog recovery procedure current.
- [ ] Provider outage and webhook-conflict procedure current.
- [ ] SSE degradation procedure current.
- [ ] Disk pressure and certificate renewal procedures current.
- [ ] Secret rotation and security incident procedures current.
- [ ] On-call/incident owner identified for release window.
- [ ] Enhanced monitoring window and rollback decision threshold agreed.

## 11. Deployment Execution

- [ ] Approved release manifest resolved.
- [ ] Backup freshness verified immediately before change.
- [ ] Maintenance mode decision made.
- [ ] Preflight and schema compatibility checks pass.
- [ ] Migration runs once and completes successfully.
- [ ] Commerce and worker become ready.
- [ ] Gateway and frontends become ready.
- [ ] Smoke tests pass.
- [ ] Tenant-isolation smoke test passes.
- [ ] Critical checkout/order/worker flow passes.
- [ ] Maintenance mode removed when safe.

## 12. Post-Deployment Validation

- [ ] Error rate and latency within expected range.
- [ ] No unusual authorization denials or cross-tenant alerts.
- [ ] PostgreSQL/Redis health normal.
- [ ] Outbox and stream lag normal.
- [ ] SSE connections stable.
- [ ] Provider interactions normal.
- [ ] No migration/backfill issue.
- [ ] Customer and tenant critical journeys verified.
- [ ] Release status communicated.

## 13. Rollback Triggers

Immediate rollback or incident escalation when any occurs:

- tenant or customer data exposure.
- incorrect financial/refund side effect.
- irreversible inventory or order corruption.
- repeated duplicate checkout/shipment/refund effects.
- migration incompatibility or widespread startup refusal.
- sustained severe error/latency/resource failure.
- unavailable recovery path.
- missing webhook authenticity enforcement.

Rollback must respect schema compatibility. A breaking migration may require forward fix or authorized restore rather than application-only rollback.

## 14. Approval Record

Required approvals:

- Product Owner: functional readiness.
- Technical Lead: architecture and implementation readiness.
- QA Lead: verification completeness.
- Security Owner: security/privacy gates.
- Operations Lead: deployment/recovery readiness.

Approval records include name, timestamp, decision, conditions, accepted risks, and expiry where applicable.

## 15. Final Decision

- [ ] GO — all mandatory gates pass.
- [ ] CONDITIONAL GO — only documented non-critical exceptions with approvals and expiry.
- [ ] NO-GO — a blocking gate remains open.

## Change Summary

Initial release-readiness checklist synchronized with the complete Terra Commerce specification and quality baseline.
