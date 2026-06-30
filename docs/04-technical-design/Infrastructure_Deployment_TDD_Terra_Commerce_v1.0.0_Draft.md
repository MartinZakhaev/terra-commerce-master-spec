# Technical Design Document — Infrastructure and Deployment

**Document ID:** TDD-TC-INF-001  
**Version:** 1.0.0  
**Status:** Draft  
**Owner:** Platform and DevOps Lead  
**Last Updated:** 2026-06-30  
**Upstream:** System Architecture v1.0.0 Final, ADR-0006 and ADR-0007, Database Design v1.0.0 Final, Security Threat Model v1.0.0 Final

## 1. Purpose

This document defines the MVP infrastructure, container topology, networking, persistent storage, configuration, secrets, CI/CD, observability, backups, deployment, rollback, capacity controls, and operational readiness for a 2 vCPU / 4 GB RAM Linux VPS.

## 2. Deployment Topology

```text
Internet
  |
Reverse Proxy :443
  |---- static frontend assets
  |---- Go gateway

Private container network
  |---- gateway
  |---- commerce
  |---- worker
  |---- postgres
  |---- redis

External services
  |---- S3-compatible object storage
  |---- Biteship
  |---- email provider
  |---- future payment provider
  |---- off-host backup storage
```

PostgreSQL and Redis are never published to the public internet.

## 3. Repository and Release Inputs

The infrastructure repository is pinned as `apps/infrastructure` in the master-spec repository.

A deployment release contains:

- gateway commit
- commerce commit
- frontend commits
- infrastructure commit
- migration version
- specification version references
- image digests
- release notes
- backup and rollback instructions

Container images must be referenced by immutable digest or immutable release tag resolved to a digest.

## 4. Container Set

### `reverse-proxy`

- TLS termination
- routing
- static assets
- security headers
- request limits
- SSE no-buffer configuration

### `gateway`

- stateless Go process
- public API and SSE
- private access to commerce, PostgreSQL where required, and Redis

### `commerce`

- stateless Medusa application
- private internal API only
- PostgreSQL and Redis access

### `worker`

- stateless background process
- PostgreSQL, Redis, object storage, and provider access

### `postgres`

- persistent volume
- no public port
- local backup client access through controlled network/process

### `redis`

- private network only
- bounded persistence and memory policy
- no public port

## 5. Network Segmentation

Recommended networks:

- `edge`: reverse proxy and gateway
- `app`: gateway, commerce, worker
- `data`: gateway where needed, commerce, worker, PostgreSQL, Redis

Rules:

- Reverse proxy cannot access PostgreSQL or Redis.
- Frontend static containers have no data-network access.
- PostgreSQL accepts only approved service identities.
- Redis accepts only gateway, commerce, and worker.
- Outbound provider access is limited to required hosts where platform tooling supports it.

## 6. Persistent Storage

Persistent:

- PostgreSQL data volume
- backup staging only when required and encrypted
- Redis AOF/RDB volume if enabled by policy

External:

- product/media objects
- generated reports
- labels where stored
- off-host database backups

Ephemeral:

- gateway, commerce, worker filesystems
- temporary report-generation files with bounded size and cleanup
- container logs after collection/rotation

No business-critical upload may exist only on local ephemeral disk.

## 7. Resource Budget

Initial memory guidance:

| Component | Limit Guidance |
|---|---:|
| Reverse proxy | 64–128 MB |
| Gateway | 192–320 MB |
| Commerce | 700–1,000 MB |
| Worker | 384–640 MB |
| PostgreSQL | 900–1,300 MB including cache behavior |
| Redis | 128–256 MB |
| OS/runtime reserve | 500–800 MB |

CPU:

- gateway responsive priority
- commerce bounded Node.js concurrency
- worker low/normal priority for reports
- PostgreSQL tuned for 2 cores

The exact allocation is validated by load tests. Containers must have limits and restart policies, but limits must not cause synchronized crash loops.

## 8. PostgreSQL Configuration

MVP tuning principles:

- conservative `max_connections`
- application connection pooling
- bounded work memory
- statement timeouts
- slow-query logging
- autovacuum monitoring
- durable WAL configuration
- timezone UTC
- backup user with limited permissions

Migrations run as a dedicated release step with a separate migration identity, not automatically and concurrently from every application replica.

## 9. Redis Configuration

- private bind/network
- authentication where supported
- explicit max memory
- eviction policy chosen to protect stream and coordination use
- stream retention policies
- AOF/RDB based on operational recovery needs
- latency and memory monitoring

Redis is not authoritative business storage.

## 10. Reverse Proxy Design

Requirements:

- TLS 1.2+ with modern ciphers
- HTTP to HTTPS redirect
- HSTS after validation
- security headers
- body-size limits by route class
- upstream connect/read/write timeouts
- SSE route with buffering and response compression disabled as needed
- trusted proxy forwarding configured explicitly
- no exposure of internal service ports

## 11. Configuration and Secrets

Configuration classes:

- non-secret environment configuration
- secret environment or mounted secret values
- global settings stored through application controls
- provider credentials

Rules:

- no secrets in repository, images, logs, or compose files committed with values
- startup validation for required configuration
- separate credentials per service and environment
- documented rotation
- restricted production access
- encrypted CI/CD secret storage

## 12. Image Build Design

- multi-stage builds
- minimal runtime images
- non-root users where practical
- read-only root filesystem where compatible
- pinned base image versions/digests
- dependency and container scanning
- build metadata and source commit labels
- no development dependencies in runtime
- health-check binaries or endpoints available

## 13. CI Pipeline

Required stages:

1. formatting and linting
2. unit tests
3. contract validation
4. operation-permission registry validation
5. cross-tenant/security tests
6. dependency and secret scanning
7. container build
8. container scan
9. integration tests
10. immutable image publication
11. release manifest generation

Protected branches and reviewed changes are required for production images.

## 14. CD and Deployment Sequence

1. Resolve approved release manifest and image digests.
2. Verify recent backup status.
3. Put system into controlled maintenance mode when migration requires it.
4. Run pre-deployment checks.
5. Apply database migrations once.
6. Start/update commerce and worker as compatible.
7. Deploy gateway.
8. Deploy static frontends.
9. Run smoke and health checks.
10. Exit maintenance mode.
11. Monitor enhanced release window.

Deployment strategy may be recreate/rolling depending on single-host capacity. Zero downtime is not guaranteed for schema-breaking MVP releases.

## 15. Migration Safety

Every migration declares:

- expected lock impact
- data backfill plan
- compatibility window
- rollback or forward-fix path
- backup implications
- expected duration

Expand-and-contract is preferred. Application code must tolerate mixed schema versions during rolling updates when rolling deployment is used.

## 16. Health Checks

Liveness checks process health only.

Readiness checks:

- required configuration
- PostgreSQL connectivity
- Redis connectivity when required for route class
- internal commerce connectivity for gateway
- migration compatibility

Dependency failure must not trigger destructive restart loops without backoff.

## 17. Backup Design

PostgreSQL:

- scheduled logical or physical backup appropriate to deployment
- encrypted transfer to off-host storage
- retention policy
- checksum/integrity validation
- backup success alert
- periodic restore test

Object storage:

- provider durability/versioning/lifecycle configuration
- private reports and labels have expiry/retention policy

Redis:

- persistence supports operational recovery but does not replace PostgreSQL/outbox recovery

## 18. Restore Procedure Design

Restore runbook must include:

1. incident and restore authorization
2. select verified restore point
3. isolate target environment
4. restore PostgreSQL
5. apply compatible migrations if required
6. validate tenant, order, inventory, shipment, audit, and outbox records
7. reconcile outbox/Redis state
8. restore object references where needed
9. smoke tests
10. controlled traffic restoration

RPO and RTO are set in operations policy and tested.

## 19. Logging and Metrics

Central collection or host-level aggregation must retain:

- application structured logs
- reverse proxy access/error logs
- PostgreSQL operational logs
- container lifecycle events
- deployment and migration logs

Metrics:

- CPU, memory, disk, inode, load
- container restarts
- HTTP latency/error
- active SSE connections
- worker and outbox lag
- PostgreSQL connections/locks/slow queries
- Redis memory/latency/stream lag
- backup status

Log rotation prevents disk exhaustion.

## 20. Alerting

Alerts include:

- service down/readiness failure
- CPU/memory/disk thresholds
- repeated restarts
- database connection saturation
- long locks/queries
- Redis memory or stream lag
- failed jobs/dead letters
- Biteship failure spike
- backup failure or stale backup
- certificate expiry
- unusual authentication or authorization failures

## 21. Security Hardening

- SSH key authentication; password login disabled where possible
- restricted administrative users
- host firewall allows required ports only
- automatic security updates according to maintenance policy
- Docker/container socket not exposed to applications
- no privileged containers unless explicitly approved
- service users non-root where practical
- production database and Redis private
- MFA for infrastructure and privileged source-control access
- deployment and secret access audited

## 22. Failure and Recovery Behavior

- Gateway failure: proxy health removes/unavailable response; restart with backoff.
- Commerce failure: gateway returns normalized unavailable errors for dependent routes.
- Worker failure: durable outbox and stream entries remain recoverable.
- PostgreSQL failure: mutations fail closed.
- Redis failure: outbox remains pending; cache/SSE/rate limit behavior follows runbook.
- Disk pressure: alerts and controlled degradation; never delete authoritative data automatically.
- Provider failure: circuit breaker/backoff and operational alerts.

## 23. Rollback Strategy

Rollback eligibility depends on schema compatibility.

- Application-only release: redeploy prior image digests.
- Backward-compatible migration: prior app may be restored if compatibility is documented.
- Breaking migration: use forward fix or database restore under incident procedure.

Every release records exact rollback steps and data-risk assessment.

## 24. Environment Strategy

At minimum:

- local development
- integration/staging
- production

Production secrets and data are never copied to lower environments without approved sanitization. Staging should exercise migrations, contracts, webhooks, SSE, backup, and restore behavior.

## 25. Capacity and Load Validation

Validate:

- 100 concurrent storefront users
- 50 concurrent SSE connections
- checkout and order load
- worker backlog recovery
- report memory behavior
- webhook bursts
- database connection limits

Test results must state resource utilization and bottlenecks on a target-equivalent host.

## 26. Operational Runbooks Required

- deployment and rollback
- migration failure
- PostgreSQL restore
- Redis outage
- outbox/worker backlog
- SSE degradation
- provider outage/webhook conflict
- secret rotation
- certificate renewal
- disk pressure
- security incident

## 27. Acceptance Criteria

1. Only proxy and intended gateway routes are public.
2. Persistent and ephemeral storage are explicit.
3. Deployment uses immutable release inputs.
4. Migrations run safely once per release.
5. Backups are off-host and restore-tested.
6. Resource limits fit the target server after load validation.
7. Secrets, networks, and service identities follow least privilege.
8. Logs, metrics, alerts, and runbooks support production operations.

## Change Summary

Initial infrastructure deployment technical design synchronized with final architecture, data, security, contracts, and release governance.
