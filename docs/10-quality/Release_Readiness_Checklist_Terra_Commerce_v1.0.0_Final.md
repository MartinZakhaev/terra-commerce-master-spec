# Release Readiness Checklist — Terra Commerce

**Document ID:** QA-TC-006  
**Version:** 1.0.0  
**Status:** Final  
**Owner:** Product Owner, Technical Lead, QA Lead, Security Owner, and Operations Lead  
**Last Updated:** 2026-06-30  
**Upstream:** All Final Terra Commerce baseline and quality documents  
**Audit:** `reviews/Quality_Test_Strategy_Suite_Audit_v1.0.0.md`

## 1. Release Identity and Provenance

- [ ] Release ID/version assigned.
- [ ] Target environment identified.
- [ ] Master-spec commit recorded.
- [ ] Application submodules pinned to exact approved commits.
- [ ] Container image digests recorded.
- [ ] Database migration version recorded.
- [ ] Gateway/commerce/worker schema compatibility ranges recorded.
- [ ] Release notes completed.
- [ ] Build provenance is available from trusted CI.
- [ ] SBOM is generated where tooling supports it.
- [ ] Artifact signature, attestation, checksum, or trusted-build verification passes.

## 2. Documentation and Traceability

- [ ] Baseline documents remain synchronized.
- [ ] Requirements Traceability Matrix updated.
- [ ] Changed mandatory requirements have implementation and evidence.
- [ ] Changed operationIds have permission mappings and contract tests.
- [ ] Changed event/SSE/webhook schemas have compatibility/consumer tests.
- [ ] ADR exists for changed architecture decisions.
- [ ] No open Critical/High document-audit finding.

## 3. Code and CI

- [ ] Required PRs reviewed and merged.
- [ ] Branch protections and checks pass.
- [ ] Format, lint, static analysis, unit, component, and integration tests pass.
- [ ] Generated artifacts match committed sources.
- [ ] Dependency, secret, and container scans pass or have approved exceptions.

## 4. Database and Migration

- [ ] Tenant impact, locks, backfill, compatibility, and data risk reviewed.
- [ ] Expand-and-contract used for destructive changes.
- [ ] Migration tested on production-like volume.
- [ ] Service schema ranges pass.
- [ ] Dedicated migration job prepared.
- [ ] Backup is fresh and integrity checked.
- [ ] Rollback or forward-fix documented.
- [ ] Restore evidence is current.

## 5. Functional Quality

- [ ] Critical E2E journeys pass.
- [ ] Applicable domain regression suites pass.
- [ ] Projection consistency checks pass.
- [ ] Dual-control workflows pass.
- [ ] Idempotency and concurrency tests pass for changed critical commands.
- [ ] No unresolved Critical defect.
- [ ] No unresolved High defect without formal exception.

## 6. Security and Privacy

- [ ] OperationId-to-permission validation passes.
- [ ] Cross-tenant/customer suite passes.
- [ ] State/version/ownership negative tests pass.
- [ ] Biteship webhook and SSE security tests pass.
- [ ] No production secret is present in source/image/log/artifact.
- [ ] Dual-control and privileged MFA requirements pass.
- [ ] Report/export permission, expiry, and audit pass.
- [ ] Critical/High threat controls have current evidence.

## 7. Performance and Capacity

- [ ] Target-host expected-load test passes.
- [ ] API/dashboard/SSE/job targets pass.
- [ ] No OOM, crash loop, sustained swap, or unsafe stop condition.
- [ ] At least 20% steady-state memory headroom.
- [ ] PostgreSQL, Redis, disk, and backlog remain safe.
- [ ] Backlog recovery is demonstrated.
- [ ] Regressions are within approved thresholds.

## 8. Infrastructure and Operations

- [ ] Only approved public routes/ports exposed.
- [ ] PostgreSQL/Redis private.
- [ ] TLS valid and monitored.
- [ ] Service credentials least privilege.
- [ ] Required configuration validated.
- [ ] Resource limits, log rotation, metrics, dashboards, and alerts active.
- [ ] Deployment, rollback, migration, restore, Redis/outbox, SSE, provider, disk, certificate, secret rotation, and security incident runbooks are current.
- [ ] Release-window incident owner identified.

## 9. Deployment Execution

- [ ] Approved manifest and artifacts resolved.
- [ ] Provenance/signature/attestation verification repeated before deployment.
- [ ] Backup freshness verified.
- [ ] Maintenance decision made.
- [ ] Preflight/schema checks pass.
- [ ] Migration runs once and succeeds.
- [ ] Commerce, worker, gateway, and frontends become ready.
- [ ] Smoke, tenant-isolation, and critical-flow checks pass.
- [ ] Enhanced monitoring begins.

## 10. Post-Deployment

- [ ] Error rate and latency normal.
- [ ] No unusual authorization or tenant alerts.
- [ ] PostgreSQL/Redis/outbox/stream/SSE/provider health normal.
- [ ] No migration/backfill issue.
- [ ] Critical customer/tenant journeys verified.
- [ ] Release status communicated.

## 11. Exceptions and Accepted Risks

Every exception must include:

- unique ID.
- affected gate/risk.
- justification and mitigation.
- accountable owner.
- approvers.
- expiry date.
- mandatory revalidation date/trigger.
- rollback/escalation condition.

Expired exceptions automatically become blocking. Conditional approval cannot remain indefinite or be copied to a later release without reapproval.

## 12. Immediate No-Go / Rollback Triggers

- Tenant/customer data exposure.
- Incorrect financial/refund effect.
- Irreversible inventory/order corruption.
- Duplicate checkout/shipment/refund effects.
- Migration incompatibility or widespread startup refusal.
- Severe sustained reliability/resource failure.
- Unavailable recovery path.
- Missing webhook authenticity enforcement.
- Failed artifact provenance verification.

## 13. Approval Record

Required approvals:

- Product Owner.
- Technical Lead.
- QA Lead.
- Security Owner.
- Operations Lead.

Records include identity, timestamp, decision, conditions, accepted-risk IDs, and expiry.

## 14. Decision

- [ ] GO — all mandatory gates pass.
- [ ] CONDITIONAL GO — only approved, owned, expiring non-critical exceptions.
- [ ] NO-GO — any blocking gate remains open.

## Change Summary

Final version closes QA-M06 by adding build provenance, SBOM/attestation verification, and mandatory ownership, expiry, and revalidation for every exception or accepted risk.
