# Performance and Load Test Plan — Terra Commerce

**Document ID:** QA-TC-005  
**Version:** 1.0.0  
**Status:** Final  
**Owner:** Platform Lead, QA Lead, and DevOps Lead  
**Last Updated:** 2026-06-30  
**Upstream:** Final NFR, Architecture, SSE Contract, Worker TDD, and Infrastructure TDD  
**Audit:** `reviews/Quality_Test_Strategy_Suite_Audit_v1.0.0.md`

## 1. Purpose

This plan verifies latency, throughput, concurrency, event propagation, backlog recovery, and resource stability on a target-equivalent 2 vCPU / 4 GB RAM host.

## 2. Baseline Targets

- Cached API p95 < 300 ms.
- Standard API p95 < 800 ms.
- Internal complex order operation < 2 seconds when not provider-bound.
- SSE propagation < 3 seconds.
- Dashboard load < 3 seconds.
- Background job start < 10 seconds.
- 100 concurrent storefront users.
- 50 concurrent SSE connections.
- No OOM, crash loop, or sustained swap.
- At least 20% steady-state memory headroom at expected peak.

Provider latency is reported separately.

## 3. Environment and Data

Use the final container topology, initial hard limits, production-like PostgreSQL/Redis settings, controlled provider simulators, and exact release image/migration/configuration identity.

Representative data includes 20 tenants, 100 tenant users, 10,000 active products, substantial historical orders, movements, tracking, audit, transitions, and outbox records. Critical query plans are captured for tenant-aware index use.

## 4. Workloads

- Storefront browsing/search/product detail/cart reads.
- Customer cart mutations, rate requests, checkout, order/tracking reads.
- Tenant order/payment/inventory/WMS/dashboard operations.
- Outbox, notification, SSE, delivery webhook, report, and reconciliation work.

## 5. Test Types

- Baseline.
- Expected load for 30–60 minutes.
- Stress to identify safe limit.
- Spike for login/checkout/webhook/SSE bursts.
- Soak for 4–8 hours.
- Worker backlog recovery.
- Dependency failure/degradation.

## 6. Critical Scenarios

1. 100 browsing users with cart mutations.
2. Checkout burst with exact-stock contention.
3. 50 SSE connections plus API traffic.
4. Duplicate/out-of-order webhook burst.
5. WMS operations during storefront load.
6. Dashboard traffic with capped report jobs.
7. Worker restart and backlog recovery.
8. Redis outage/recovery.
9. PostgreSQL saturation and timeout behavior.
10. Deployment warm-up and recovery.

## 7. Metrics

Capture API latency/throughput/errors, provider wait, state conflicts, SSE connections/disconnects, outbox age/count, stream lag, retry/dead letters, event-to-SSE time, CPU, memory, swap, disk, restarts, PostgreSQL connections/locks/slow queries, Redis memory/latency/evictions, and report duration/size.

## 8. Pass Criteria

Expected peak passes when:

- latency targets hold for at least 95% of stable window.
- unexplained error rate remains below 1%.
- no tenant, security, state, financial, inventory, shipment, or event integrity failure occurs.
- no OOM or restart loop.
- memory headroom remains at least 20%.
- swap is not sustained.
- backlog remains bounded or recovers in the documented window.
- PostgreSQL and Redis remain within safe limits.

Any duplicate financial, inventory, shipment, or event side effect automatically fails the run.

## 9. Immediate Stop Conditions

Abort the test and preserve evidence when any occurs:

- OOM kill or repeated crash loop.
- sustained swap with rapidly worsening latency.
- disk/inode usage approaches configured critical threshold or uncontrolled log/stream growth.
- PostgreSQL connection exhaustion, prolonged blocking lock, or unsafe replication/WAL/disk condition.
- runaway outbox/Redis Stream growth beyond the recovery envelope.
- integrity anomaly, tenant leakage, duplicate financial/inventory/shipment effect, or unexpected destructive transition.
- monitoring failure that makes results untrustworthy.

The environment is stabilized and root cause reviewed before resuming.

## 10. Regression Thresholds

Flag regression when p95 worsens >15%, throughput falls >10%, memory rises >15%, recovery time rises >20%, or critical query plan loses its tenant-aware index under equivalent conditions.

## 11. Provider Simulation

Simulators support normal, slow, timeout, rate limit, 5xx, success-after-timeout, duplicate webhook, and out-of-order webhook behavior.

## 12. Execution and Evidence

Validate telemetry, seed data, warm as required, run baseline/load/stress/spike/backlog/failure/soak, analyze, tune, and re-run.

Evidence includes scripts, environment identity, latency/throughput, resource graphs, queue/error analysis, stop events, bottlenecks, comparison with prior baseline, and final decision.

## 13. Acceptance Criteria

1. API, SSE, worker, webhook, and report workloads are covered.
2. Targets and stop conditions are explicit.
3. Integrity failures override performance success.
4. Regression is measurable.
5. Results are tied to exact release artifacts.

## Change Summary

Final version closes QA-M05 by adding immediate database, disk, event-backlog, swap, OOM, monitoring, and integrity stop conditions.
