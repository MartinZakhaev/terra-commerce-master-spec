# Deployment and Rollback Runbook — Terra Commerce

**Document ID:** OPS-TC-001  
**Version:** 1.0.0  
**Status:** Draft  
**Owner:** Operations and DevOps Lead  
**Last Updated:** 2026-07-01  
**Upstream:** Infrastructure TDD, Release Readiness Checklist, Environment Specification, and Quality Strategy

## 1. Purpose

Provides the executable procedure for production deployment, migration, verification, rollback, and release communication.

## 2. Preconditions

- Approved release-readiness checklist.
- Exact master-spec and submodule commits.
- Verified image digests and provenance.
- Current backup and restore evidence.
- Schema compatibility declarations.
- Migration and rollback/forward-fix plan.
- Named release commander and incident owner.
- Monitoring and alerting operational.

## 3. Roles

- Release Commander: coordinates decision and timeline.
- Migration Operator: executes migration once.
- Application Operator: deploys services.
- QA Verifier: runs smoke/critical tests.
- Security/Operations Observer: watches anomalies.
- Product Approver: validates business readiness.

## 4. Preflight

1. Confirm maintenance window and communication.
2. Record current release manifest and health baseline.
3. Verify disk, memory, database, Redis, outbox, and backup status.
4. Confirm configuration and secret references.
5. Validate schema compatibility ranges.
6. Verify rollback artifacts remain available.
7. Pause risky manual operations if required.

## 5. Deployment Procedure

1. Enable maintenance mode when migration requires it.
2. Stop or drain workers if migration compatibility requires it.
3. Run dedicated migration job once.
4. Verify migration status and constraints.
5. Deploy commerce.
6. Deploy worker and verify no unsafe backlog processing.
7. Deploy gateway.
8. Deploy frontends/static assets.
9. Verify readiness and release versions.
10. Run smoke tests: health, tenant resolution, login, catalog, cart, checkout simulation, order read, worker/outbox, SSE, webhook simulator.
11. Run tenant-isolation smoke tests.
12. Remove maintenance mode.
13. Start enhanced monitoring window.

## 6. Validation

Check:

- error and latency baseline.
- database locks/connections.
- Redis memory/stream lag.
- outbox age and worker failures.
- SSE connection stability.
- provider calls.
- critical business projections and audit events.
- no unusual authorization denials.

## 7. Rollback Decision

Rollback or escalate immediately for tenant leakage, financial duplication/corruption, migration incompatibility, irreversible inventory/order corruption, widespread startup failure, severe sustained resource failure, or absent recovery path.

## 8. Rollback Paths

### Application-only

Deploy prior image digests when schema remains compatible.

### Backward-compatible migration

Revert application and leave expanded schema when documented safe.

### Breaking migration

Do not blindly redeploy old code. Use approved forward fix or database restore under incident control.

### Configuration

Restore prior approved configuration version/reference and restart/reload as documented.

## 9. Rollback Procedure

1. Declare rollback and notify stakeholders.
2. Re-enter maintenance mode if needed.
3. Stop new mutations/work claims.
4. Preserve logs, metrics, and migration evidence.
5. Execute selected rollback path.
6. Verify schema/service compatibility.
7. Run smoke, tenant-isolation, and data-integrity checks.
8. Reconcile outbox/Redis/provider uncertainty.
9. Restore traffic gradually.
10. Record outcome and open follow-up review.

## 10. Post-Release

- Confirm monitoring window completion.
- Record final release status.
- Attach evidence and deviations.
- Close or extend accepted risks.
- Schedule retrospective for incidents or rollback.

## 11. Abort Conditions

Abort before mutation when backup, schema compatibility, provenance, telemetry, disk headroom, or required approver is missing.

## 12. Acceptance Criteria

1. Deployment is reproducible from immutable artifacts.
2. Migration runs once.
3. Rollback respects schema compatibility.
4. Tenant and critical-flow validation occurs before release completion.
5. Evidence and decisions are recorded.

## Change Summary

Initial deployment and rollback runbook draft aligned with finalized infrastructure and quality controls.
