# Deployment and Rollback Runbook — Terra Commerce

**Document ID:** OPS-TC-001  
**Version:** 1.0.0  
**Status:** Final  
**Owner:** Operations and DevOps Lead  
**Last Updated:** 2026-07-01  
**Upstream:** Infrastructure TDD, Release Readiness Checklist, Environment Specification, Quality Strategy  
**Audit:** `../09-infrastructure/reviews/Infrastructure_Operations_Suite_Audit_v1.0.0.md`

## 1. Purpose

Provides the executable procedure for production deployment, migration, verification, rollback, and release communication.

## 2. Preconditions

- Approved release-readiness checklist.
- Exact master-spec/submodule commits and image digests.
- Verified provenance and configuration fingerprint.
- Current backup and restore evidence.
- Declared service/schema compatibility.
- Migration and rollback/forward-fix plan.
- Named release commander, migration operator, QA verifier, and incident owner.
- Critical telemetry available.

## 3. Preflight

1. Confirm window and stakeholders.
2. Capture current manifest and health baseline.
3. Verify disk, memory, PostgreSQL, Redis, outbox, worker, and backup status.
4. Confirm configuration/secret references and rollback artifacts.
5. Validate schema ranges and migration duration/lock expectations.
6. Confirm worker-drain plan and current active leases/claims.
7. Abort if any mandatory prerequisite is missing.

## 4. Worker Drain

When required:

- stop new claims and scheduler ownership.
- wait for active jobs or safely release leases.
- verify pending acknowledgements and outbox claims.
- record remaining pending work.
- ensure no provider call is left in an untracked uncertain state.

Do not terminate workers until claim/lease state is understood.

## 5. Deployment Procedure

1. Enable maintenance mode if needed.
2. Drain workers according to Section 4.
3. Run the dedicated migration job once.
4. Verify migration status, constraints, and schema compatibility.
5. Deploy commerce.
6. Deploy worker in controlled concurrency and verify backlog behavior.
7. Deploy gateway.
8. Deploy frontends/static assets.
9. Verify readiness and release versions.
10. Run smoke, tenant-isolation, and critical-flow tests.
11. Remove maintenance mode.
12. Start enhanced monitoring.

## 6. Point of No Return

The release plan explicitly identifies the point after which application-only rollback is unsafe, such as irreversible data transformation, removal of old schema, or provider-side one-way change.

Crossing this point requires explicit authorization from Release Commander, Migration Operator, Technical Lead, and Operations Lead after confirming backup, forward-fix, restore, and communication readiness.

Before this point, use the documented application/configuration rollback path. After it, use approved forward fix or disaster recovery; do not deploy incompatible old binaries.

## 7. Validation

Check errors/latency, database locks/connections, Redis/stream lag, outbox age, dead letters, SSE stability, provider outcomes, projections, audit events, and authorization anomalies.

## 8. Rollback Triggers

Tenant/customer leakage, financial duplication/corruption, inventory/order corruption, migration incompatibility, startup refusal, severe resource failure, missing webhook security, or unavailable recovery path.

## 9. Rollback Paths

- Application-only: previous image digests when schema is compatible.
- Expanded backward-compatible schema: revert app and retain safe additions.
- Breaking migration: forward fix or authorized restore.
- Configuration: restore previous approved reference/version.

## 10. Rollback Procedure

1. Declare rollback and notify.
2. Enter maintenance mode if required.
3. Stop new mutations and work claims.
4. Preserve logs, metrics, claims, and migration evidence.
5. Execute selected path.
6. Verify schema/service/config compatibility.
7. Reconcile outbox, Redis, payment, and shipment uncertainty.
8. Run smoke, tenant-isolation, and integrity checks.
9. Restore traffic gradually.
10. Record outcome and follow-up actions.

## 11. Post-Release

Record final status, evidence, deviations, exceptions, and monitoring results. Schedule retrospective for rollback, incident, or unexpected degradation.

## 12. Acceptance Criteria

1. Deployment is reproducible from immutable artifacts.
2. Worker drain and claims are verified.
3. Migration runs once.
4. Point of no return is explicit and authorized.
5. Rollback respects schema and provider compatibility.
6. Tenant/integrity validation occurs before completion.

## Change Summary

Final version closes OPS-M04 by adding worker-drain verification and an explicitly authorized deployment point of no return.
