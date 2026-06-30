# Performance and Load Test Plan — Terra Commerce

**Document ID:** QA-TC-005  
**Version:** 1.0.0  
**Status:** Draft  
**Owner:** Platform Lead, QA Lead, and DevOps Lead  
**Last Updated:** 2026-06-30  
**Upstream:** PRD/FSD NFRs, Final Architecture, TDD Suite, SSE Contract, Worker/Event Pipeline TDD, and Infrastructure Deployment TDD

## 1. Purpose

This plan verifies that Terra Commerce meets response-time, throughput, concurrency, backlog-recovery, and resource-stability expectations on a target-equivalent 2 vCPU / 4 GB RAM Linux host.

## 2. Baseline Targets

- Cached API p95 below 300 ms.
- Standard API p95 below 800 ms.
- Internal complex order operation below 2 seconds when not provider-bound.
- SSE propagation below 3 seconds.
- Typical dashboard load below 3 seconds.
- Background job start within 10 seconds.
- 100 concurrent storefront users.
- 50 concurrent SSE connections.
- 1,000 daily platform orders as baseline planning volume.
- No OOM kills, crash loops, or sustained swap usage.
- At least 20% steady-state memory headroom under expected peak.

Provider-dependent latency is measured separately from internal processing.

## 3. Test Environment

- Target-equivalent 2 vCPU / 4 GB RAM host.
- Same container topology and initial hard limits as final Infrastructure TDD.
- PostgreSQL and Redis configured with production-like limits.
- External providers replaced by latency/error-controlled simulators unless sandbox behavior is explicitly measured.
- Representative data volume: tenants, users, 10,000 active products, orders, inventory, tracking history, audit, and outbox history.

All test runs record exact image digests, migration version, configuration, dataset, and host metrics.

## 4. Workload Models

### Storefront Read Mix

- Product list/search/filter.
- Product detail.
- Category list.
- Cart reads.

### Customer Mutation Mix

- Cart create/add/update/remove.
- Login and address reads.
- Shipping-rate requests.
- Checkout.
- Order and tracking reads.

### Tenant Operations Mix

- Order lists/details.
- Payment confirmation.
- Inventory reads and adjustments.
- Picking/packing transitions.
- Dashboard/report requests.

### Event and Worker Mix

- Outbox publication.
- Notification projection.
- SSE projection.
- Delivery webhook bursts.
- Report generation.
- Backlog recovery.

## 5. Test Types

### Baseline Test

Single-user and low-concurrency response times establish implementation overhead and detect obvious regressions.

### Load Test

Expected peak workload for at least 30–60 minutes to establish stable p50/p95/p99 latency, throughput, errors, and resource usage.

### Stress Test

Increase load until service-level objectives fail, then identify bottleneck and safe operating limit.

### Spike Test

Sudden bursts of login, checkout, webhook, or SSE connection attempts.

### Soak Test

Extended 4–8 hour workload to detect memory leaks, connection leaks, stream growth, queue accumulation, and disk pressure.

### Backlog Recovery Test

Stop or throttle workers, accumulate outbox/stream backlog, restore capacity, and measure recovery without duplicate side effects.

### Failure-Degradation Test

Simulate Redis, provider, worker, or partial database pressure and verify bounded failure, recovery, and no state corruption.

## 6. Critical Scenarios

1. 100 storefront users browsing with mixed cart mutations.
2. Checkout bursts with inventory contention.
3. 50 concurrent SSE connections plus normal API traffic.
4. Webhook burst with duplicate and out-of-order events.
5. Picking/packing operations while storefront traffic continues.
6. Dashboard reads while report jobs run at capped concurrency.
7. Worker restart with outbox and pending-stream recovery.
8. Redis outage and restoration.
9. PostgreSQL connection saturation and statement timeout behavior.
10. Rolling/recreate deployment recovery and warm-up.

## 7. Data and Query Scale

Test datasets include:

- 20 active tenants.
- 100 tenant admin users.
- 10,000 active products platform-wide.
- At least 100,000 historical orders for query/index evaluation when feasible.
- Inventory movement, tracking, audit, and transition history sufficient to exercise pagination and retention indexes.

Query plans for critical endpoints are captured and checked for tenant-aware index use.

## 8. Metrics

Application:

- request throughput
- p50/p95/p99 latency
- error and timeout rate
- state conflicts
- provider wait time
- active SSE connections and disconnects

Worker/event:

- outbox pending count and oldest age
- publication rate/failure
- stream lag and pending entries
- event-to-SSE propagation
- retry/dead-letter count
- backlog recovery time

Infrastructure:

- CPU, memory, swap, disk, inode, load
- container restarts/OOM
- PostgreSQL connections, locks, cache, slow queries, WAL/checkpoint behavior
- Redis memory, latency, evictions, stream size
- network and object-storage transfer

## 9. Pass/Fail Criteria

Expected-peak load passes when:

- latency targets are met for at least 95% of the stable test window.
- error rate remains below 1% excluding intentional provider failures, with no unexplained 5xx spike.
- no tenant/security/state-integrity failure occurs.
- no OOM or crash loop occurs.
- steady-state host memory headroom remains at least 20%.
- swap is not sustained.
- queue lag remains bounded or recovers within the documented window.
- PostgreSQL and Redis remain within configured safe limits.

Any duplicate financial, inventory, shipment, or event side effect is an automatic failure regardless of latency.

## 10. Regression Thresholds

A release is flagged when, under the same controlled workload:

- p95 latency worsens by more than 15%.
- throughput falls by more than 10%.
- memory grows by more than 15% without explained workload change.
- worker recovery time increases by more than 20%.
- database query plan loses the intended tenant-aware index.

Threshold exceptions require documented cause and approval.

## 11. Provider Simulation

Simulators support configurable:

- normal latency.
- slow response.
- timeout.
- rate limit.
- 5xx error.
- success after uncertain timeout.
- duplicate webhook.
- out-of-order webhook.

Internal latency and provider latency are reported separately.

## 12. Execution Phases

1. Validate environment and telemetry.
2. Seed dataset.
3. Warm application and caches where test requires it.
4. Run baseline.
5. Run expected load.
6. Run stress/spike.
7. Run backlog/failure scenarios.
8. Run soak before major release.
9. Analyze bottlenecks and regressions.
10. Re-run after tuning.

## 13. Evidence

Required output:

- workload definition and scripts.
- release and environment identity.
- latency/throughput summaries.
- host/container/database/Redis graphs.
- error and queue analysis.
- bottleneck findings.
- comparison with prior baseline.
- pass/fail decision and approved exceptions.

## 14. Acceptance Criteria

1. Target-host tests cover API, SSE, worker, webhook, and reporting workloads.
2. Resource and latency targets are measurable.
3. Integrity failures automatically fail the run.
4. Regression thresholds are explicit.
5. Results are tied to exact release artifacts.

## Change Summary

Initial performance and load test plan aligned with the MVP capacity, latency, resource, and recovery requirements.
