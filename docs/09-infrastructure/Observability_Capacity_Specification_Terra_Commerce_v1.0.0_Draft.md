# Observability and Capacity Specification — Terra Commerce

**Document ID:** INF-TC-002  
**Version:** 1.0.0  
**Status:** Draft  
**Owner:** Platform, DevOps, and Operations Lead  
**Last Updated:** 2026-07-01  
**Upstream:** Final Architecture, Worker/Event TDD, Infrastructure TDD, Threat Model, Performance Plan, and Release Checklist

## 1. Purpose

Defines logs, metrics, traces, dashboards, alerts, service-level indicators, capacity thresholds, retention, privacy, escalation, and scaling triggers for Terra Commerce MVP.

## 2. Observability Principles

- Every request and event is correlatable.
- Tenant context is included where appropriate but personal data is minimized.
- Alerts are actionable and owned.
- PostgreSQL remains the source of business truth.
- Monitoring failure is itself observable.
- Capacity decisions use measured trends, not intuition.

## 3. Correlation Model

Required identifiers:

- correlation ID
- operation ID
- tenant ID where applicable
- pseudonymous principal ID/type
- aggregate/resource ID
- event ID and aggregate version
- provider event/order reference
- release and service version

## 4. Logging Standard

Structured JSON logs include timestamp, severity, service, environment, release, operation, correlation, tenant, outcome, latency, and sanitized error category.

Prohibited:

- passwords, tokens, secret headers, payment instruments
- raw provider secrets
- unnecessary full addresses or personal data
- unsanitized request/response bodies

Log levels are centrally defined. Debug logging is disabled in production unless time-bounded and approved.

## 5. Metrics Catalog

### Gateway/API

- request count, p50/p95/p99 latency, 4xx/5xx
- authorization denials
- rate-limit events
- idempotency replay/conflict
- state conflicts
- downstream timeout/error
- active SSE connections, reconnects, replay misses, slow-consumer disconnects

### Commerce

- checkout success/failure
- payment transition outcomes
- inventory reservation contention
- picking/packing throughput and exceptions
- shipment creation/reconciliation
- refund outcomes

### Worker/Event

- outbox pending count and oldest age
- publish rate/failure
- stream length, lag, pending entries
- retries, version gaps, dead letters
- notification/report/delivery outcomes

### Data/Infrastructure

- CPU, memory, swap, disk, inode, load
- container restarts/OOM
- PostgreSQL connections, locks, query latency, cache, WAL/checkpoints, storage
- Redis memory, latency, evictions, persistence, stream growth
- backup age/success/restore-test age
- certificate expiry

## 6. Tracing

Distributed tracing is optional for MVP but recommended for gateway-to-commerce, provider calls, checkout, webhook processing, and outbox-to-consumer paths. Trace sampling must avoid excessive cost and sensitive payload capture.

## 7. Dashboards

Required dashboards:

1. Platform health overview.
2. API and customer journey health.
3. Checkout/payment/order funnel.
4. Inventory and WMS operations.
5. Delivery/Biteship integration.
6. Outbox, streams, workers, and dead letters.
7. PostgreSQL/Redis/host capacity.
8. Security and authorization anomalies.
9. Backup and recovery readiness.
10. Release comparison dashboard.

## 8. Alert Severity

- SEV-1: tenant leakage, financial corruption, total outage, unrecoverable data risk.
- SEV-2: critical journey unavailable, major provider/DB/worker failure, resource exhaustion imminent.
- SEV-3: degraded service, growing backlog, elevated errors, backup staleness.
- SEV-4: warning/trend requiring planned action.

Every alert has owner, runbook, threshold, evaluation window, deduplication, and escalation route.

## 9. Initial Alert Thresholds

Examples subject to tuning:

- API 5xx > 2% for 5 minutes.
- p95 standard API > 1.2 seconds for 10 minutes.
- memory > 80% for 10 minutes; > 90% immediate SEV-2.
- sustained swap activity.
- disk > 80% warning, > 90% urgent.
- PostgreSQL connections > 80% pool capacity.
- long lock/query over configured threshold.
- outbox oldest pending > 60 seconds warning, > 5 minutes urgent.
- dead-letter count > 0 for critical consumers.
- Redis memory > 80% or unexpected eviction.
- backup stale beyond schedule or restore test overdue.
- certificate expiry within 30/14/7 days.

## 10. Capacity Baseline

Initial target host: 2 vCPU / 4 GB RAM.

Service hard limits follow Infrastructure TDD. Expected peak qualification requires:

- 100 concurrent storefront users.
- 50 SSE connections.
- at least 20% steady-state memory headroom.
- no sustained swap.
- bounded worker backlog.
- no OOM or restart loop.

## 11. Capacity Forecasting

Track weekly/monthly trends for:

- tenant/user/product/order growth
- request volume and concurrency
- database size and index growth
- audit/tracking/outbox retention
- object storage
- SSE connections
- provider requests
- worker throughput/backlog

Forecast at least 30, 60, and 90 days where enough history exists.

## 12. Scaling Triggers

Trigger investigation when any persists:

- memory headroom below 25% at normal peak.
- CPU above 70% sustained.
- PostgreSQL connection/IO/lock saturation.
- worker backlog fails to recover within target.
- SSE limits approached regularly.
- p95 degradation after tuning.
- disk growth threatens retention window.

Scaling sequence follows final architecture: tune, externalize storage, separate PostgreSQL, separate Redis, add workers, scale gateway/commerce, then evaluate extraction by ADR criteria.

## 13. Retention

- Operational logs retained according to security/operations policy.
- Critical audit evidence follows longer policy.
- High-cardinality metrics and traces use bounded retention.
- Personal data is minimized and removed according to retention rules.
- Monitoring evidence for releases is retained according to Quality Strategy.

## 14. Synthetic and Business Checks

Synthetic checks cover health, tenant storefront browse, authentication, cart, checkout simulation where safe, SSE connect, and provider adapter health.

Business checks detect impossible trends such as orders without reservations, shipments without packing completion, refunded amount above captured, or outbox backlog with no publisher.

## 15. Monitoring Failure

Alert when collectors, dashboards, exporters, or log pipelines stop reporting. Release validation must not proceed when critical telemetry is unavailable.

## 16. Acceptance Criteria

1. Every critical service and workflow has measurable signals.
2. Alerts have owners and runbooks.
3. Capacity thresholds align with performance and infrastructure documents.
4. Sensitive data is excluded.
5. Scaling triggers follow approved architecture.

## Change Summary

Initial observability and capacity specification draft synchronized with finalized technical, security, quality, and infrastructure baselines.
