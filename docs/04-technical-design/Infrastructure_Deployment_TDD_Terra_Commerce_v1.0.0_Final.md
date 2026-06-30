# Technical Design Document — Infrastructure and Deployment

**Document ID:** TDD-TC-INF-001  
**Version:** 1.0.0  
**Status:** Final  
**Owner:** Platform and DevOps Lead  
**Last Updated:** 2026-06-30  
**Upstream:** Final Architecture, ADRs, Database Design, Contract Suite, Threat Model  
**Audit:** `reviews/Technical_Design_Suite_Audit_v1.0.0.md`

## 1. Topology

Public traffic reaches only the reverse proxy on HTTPS and intended gateway routes. Gateway, commerce, worker, PostgreSQL, and Redis communicate over private networks. PostgreSQL and Redis are never publicly exposed.

External dependencies are S3-compatible object storage, Biteship, email, future payment provider, and off-host backup storage.

## 2. Release Composition

Each release records exact submodule commits, image digests, migration version, specification versions, infrastructure commit, release notes, backup implications, and rollback procedure.

Images are immutable, built from reviewed commits, scanned, and deployed by digest or immutable tag resolved to a digest.

## 3. Containers and Networks

Containers:

- reverse proxy
- Go gateway
- Medusa commerce
- worker
- PostgreSQL
- Redis

Networks:

- edge: proxy and gateway
- app: gateway, commerce, worker
- data: approved application services, PostgreSQL, Redis

Proxy cannot access the data network. Static frontends have no database/Redis connectivity. Service identities are separate and least privilege.

## 4. Storage

Persistent:

- PostgreSQL volume
- Redis persistence volume when enabled
- encrypted temporary backup staging only when required

External:

- tenant media
- reports and labels
- off-host backups

Gateway, commerce, and worker filesystems are replaceable. Business-critical files must not exist only on ephemeral disk.

## 5. Conservative Initial Resource Profile

The initial hard-limit profile must leave operating-system, kernel, filesystem-cache, and deployment headroom:

| Component | Initial memory limit |
|---|---:|
| Reverse proxy | 96 MB |
| Gateway | 256 MB |
| Commerce | 768 MB |
| Worker | 384 MB |
| PostgreSQL container/process envelope | 1,024 MB |
| Redis | 192 MB |
| Reserved for OS, cache, deployment variance | approximately 1,376 MB |

Total configured application/data-service limits are approximately 2.72 GB, leaving roughly 1.28 GB on a 4 GB host before implementation-specific overhead adjustments.

Limits may increase only after target-host load tests demonstrate sustained memory headroom, acceptable swap behavior, and no OOM/restart instability.

Swap may be configured as encrypted emergency protection against abrupt OOM, with conservative swappiness and alerts. Swap is not counted as normal capacity and sustained swapping fails performance acceptance.

## 6. PostgreSQL and Redis

PostgreSQL uses conservative connections, application pooling, bounded work memory, statement timeouts, slow-query monitoring, durable WAL settings, UTC, and a dedicated migration/backup identity.

Redis is private, authenticated where supported, memory bounded, monitored, and configured with an eviction/persistence policy that protects Streams and coordination data. Redis never replaces PostgreSQL as business truth.

## 7. Reverse Proxy

- TLS 1.2+.
- HTTPS redirect.
- HSTS after validation.
- Security headers.
- Route-specific body limits and timeouts.
- SSE buffering disabled only on stream route.
- Explicit trusted proxy configuration.
- No internal port exposure.

## 8. Configuration and Secrets

Secrets are not stored in repositories, images, committed compose files, logs, or public application settings. Each service/environment uses separate credentials. CI/CD secret storage is encrypted and production access is restricted and audited.

Startup validates all required configuration and refuses insecure defaults.

## 9. Build and CI

Images use multi-stage builds, minimal runtime layers, non-root users where practical, pinned bases, source metadata, and no development dependencies.

CI stages:

1. format/lint
2. unit tests
3. contract validation
4. operation-permission registry validation
5. cross-tenant/security tests
6. dependency/secret scans
7. image build and scan
8. integration tests
9. immutable publication
10. release manifest generation

## 10. Schema Compatibility Contract

Every gateway, commerce, and worker release declares:

- minimum supported schema version
- maximum supported schema version
- required migration state
- backward/forward compatibility window

Readiness fails and startup refuses service when the current schema is outside the declared range.

Migration status is stored and checked before traffic/work claiming. Only the dedicated release migration job applies migrations.

Destructive changes require a completed expand-and-contract sequence:

1. expand schema compatibly
2. deploy code supporting old/new forms
3. backfill and verify
4. switch reads/writes
5. remove old form in a later release

A single release must not combine an irreversible destructive migration with code that has no verified rollback/forward-fix path.

## 11. Deployment Sequence

1. Resolve approved manifest and digests.
2. Verify backup freshness and integrity.
3. Validate schema compatibility declarations.
4. Enter maintenance mode when required.
5. Run preflight checks.
6. Apply migrations once.
7. Verify migration status.
8. Deploy compatible commerce and worker.
9. Deploy gateway and frontends.
10. Run readiness, smoke, tenant-isolation, and critical-flow checks.
11. Exit maintenance mode.
12. Monitor the release window.

Single-host capacity may require controlled recreate deployment; zero downtime is not guaranteed for incompatible MVP migrations.

## 12. Health and Restart Behavior

Liveness measures process health. Readiness validates configuration, schema compatibility, and route-critical dependencies.

Restarts use backoff and bounded attempts to avoid synchronized crash loops. Dependency loss must not trigger destructive automatic actions.

## 13. Backup and Restore

PostgreSQL backups are encrypted, integrity checked, stored off-host, retained by policy, and alerted on failure/staleness. Restore tests verify tenant, order, inventory, shipment, audit, transition, and outbox records.

Restore procedure includes authorization, isolated target, restore point selection, migration compatibility, domain validation, Redis/outbox reconciliation, smoke tests, and controlled traffic restoration.

RPO/RTO are defined and tested in operations policy.

## 14. Observability and Alerts

Collect structured application/proxy/database/container/deployment logs with rotation.

Metrics include host/container resources, restarts, HTTP latency/errors, SSE connections, worker/outbox lag, PostgreSQL connections/locks/queries, Redis memory/latency, backup status, and provider failures.

Alerts include service failure, resource pressure, database saturation, stream lag, dead letters, backup failure, certificate expiry, security anomalies, and repeated authorization failures.

## 15. Security Hardening

- SSH keys and restricted administrators.
- Host firewall with minimal ports.
- Non-privileged containers and no application access to container socket.
- Private data services.
- MFA for infrastructure/source-control privileged access.
- Controlled security updates.
- Deployment and secret access audit.
- Lower environments never receive unsanitized production data.

## 16. Failure and Rollback

Gateway/commerce failures return normalized unavailable responses. Worker failure remains recoverable from outbox/Streams. PostgreSQL mutations fail closed. Redis failure leaves outbox pending and follows degraded-service runbooks. Disk pressure alerts before exhaustion and never triggers deletion of authoritative data.

Application rollback redeploys prior digests only when schema compatibility permits. Breaking migration failure uses forward fix or authorized restore. Every release includes a data-risk assessment.

## 17. Capacity Validation

Target-host tests cover 100 storefront users, 50 SSE connections, checkout/order flows, webhook bursts, backlog recovery, reports, and connection limits.

Promotion criteria:

- no OOM kills or restart loops
- no sustained swap activity
- minimum 20% steady-state memory headroom under expected peak
- documented CPU and database bottlenecks
- performance targets satisfied
- recovery from burst/backlog demonstrated

## 18. Required Runbooks

Deployment/rollback, migration failure, PostgreSQL restore, Redis outage, outbox backlog, SSE degradation, provider outage, webhook conflict, secret rotation, certificate renewal, disk pressure, and security incident.

## 19. Acceptance Criteria

1. Public/private network boundaries are enforced.
2. Initial hard limits fit safely inside 4 GB.
3. Swap is emergency-only.
4. Services declare and enforce schema compatibility.
5. Destructive changes use expand-and-contract.
6. Backups are off-host and restore-tested.
7. Immutable releases, observability, alerts, and runbooks are defined.

## Change Summary

Final version closes TDD-M05 and TDD-M06 with a conservative 4 GB hard-limit profile, emergency-only swap policy, load-test promotion gates, and explicit service/schema compatibility and expand-contract deployment rules.
