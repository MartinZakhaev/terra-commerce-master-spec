# Observability and Capacity Specification — Terra Commerce

**Document ID:** INF-TC-002  
**Version:** 1.0.0  
**Status:** Final  
**Owner:** Platform, DevOps, and Operations Lead  
**Last Updated:** 2026-07-01  
**Upstream:** Final Architecture, Worker/Event TDD, Infrastructure TDD, Threat Model, Performance Plan, Release Checklist  
**Audit:** `reviews/Infrastructure_Operations_Suite_Audit_v1.0.0.md`

## 1. Purpose

Defines logs, metrics, traces, dashboards, service-level indicators, alerts, capacity forecasts, retention, privacy, escalation, and scaling triggers.

## 2. Principles

- Requests, events, jobs, and provider callbacks are correlatable.
- Tenant context is included where useful and personal data is minimized.
- Alerts are actionable, owned, linked to runbooks, and resistant to flapping.
- Monitoring failure is observable.
- Capacity decisions are based on measured trends and forecasts.

## 3. Correlation and Logging

Required identifiers include correlation ID, operation ID, tenant ID where applicable, pseudonymous principal, aggregate/resource ID, event ID/version, provider reference, and release/service version.

Logs are structured JSON. Passwords, tokens, secret headers, payment instruments, full unnecessary addresses, and unsanitized bodies are prohibited. Production debug logging is time-bounded and approved.

## 4. Metrics

### Gateway and SSE

Request rate/latency/errors, authorization denials, rate limits, idempotency results, state conflicts, downstream failures, SSE connections/replays/disconnects.

### Commerce and WMS

Checkout/payment outcomes, reservation contention, picking/packing throughput, exceptions, shipment/refund outcomes.

### Worker and Events

Outbox age/count, publication rate/failure, stream lag/pending, retries, version gaps, dead letters, notification/report/delivery outcomes.

### Data and Host

CPU, memory, swap, disk, inode, restarts/OOM, PostgreSQL connections/locks/query latency/WAL/checkpoints/storage, Redis memory/latency/evictions/persistence, backup/restore age, certificate expiry.

## 5. Dashboards

Required dashboards:

1. Platform health.
2. API/customer journeys.
3. Checkout/payment/order funnel.
4. Inventory/WMS.
5. Delivery/Biteship.
6. Outbox/streams/workers.
7. PostgreSQL/Redis/host capacity.
8. Security/authorization anomalies.
9. Backup/recovery readiness.
10. Release comparison.

## 6. Alert Design

Every paging alert includes severity, owner, direct runbook link, threshold or SLO rule, evaluation window, deduplication key, maintenance behavior, and escalation path.

Use multi-window burn-rate alerts where an SLO exists, combining fast and slow windows to detect severe and sustained degradation. Static thresholds use hysteresis, cooldown, and recovery criteria to prevent flapping. Planned maintenance suppresses only expected alerts and never hides security or data-integrity signals.

## 7. Severity

- SEV-1: tenant leak, financial/data corruption, total outage, unrecoverable risk.
- SEV-2: critical journey unavailable, major dependency failure, imminent exhaustion.
- SEV-3: degraded service, growing backlog, elevated errors, backup staleness.
- SEV-4: warning/trend requiring planned action.

## 8. Initial Thresholds

Examples subject to measured tuning:

- API 5xx >2% for 5 minutes.
- Standard API p95 >1.2 seconds for 10 minutes.
- Memory >80% warning, >90% urgent.
- Sustained swap activity.
- Disk >80% warning, >90% urgent.
- PostgreSQL connections >80% pool capacity.
- Long lock/query beyond configured threshold.
- Outbox oldest >60 seconds warning, >5 minutes urgent.
- Any critical-consumer dead letter.
- Redis memory >80% or unexpected eviction.
- Backup/restore validation overdue.
- Certificate expiry at 30/14/7 days.

## 9. Capacity Baseline

Target host: 2 vCPU / 4 GB RAM. Qualification requires 100 storefront users, 50 SSE connections, at least 20% steady-state memory headroom, no sustained swap, bounded backlog, and no OOM/restart loop.

## 10. Forecasting and Storage Gates

Track 30/60/90-day forecasts for tenants, users, products, orders, request concurrency, database size, indexes, WAL, backup volume/duration, audit/transition/tracking/outbox retention, Redis streams, object storage, and provider traffic.

Capacity review is mandatory when forecast shows:

- PostgreSQL volume or filesystem reaching 70% within 60 days or 80% within 30 days.
- WAL/checkpoint or backup growth exceeding backup window or off-host retention budget.
- Audit/tracking/event retention exceeding partition/archive targets.
- Object-storage growth or cost exceeding approved budget threshold.
- Disk growth reducing restore or migration headroom.

## 11. Scaling Triggers and Order

Investigate persistent CPU >70%, memory headroom <25%, database saturation, backlog recovery failure, regular SSE-limit pressure, p95 degradation, or unsafe disk forecast.

Scaling order: tune, externalize storage, separate PostgreSQL, separate Redis, add workers, scale gateway/commerce, then evaluate extraction under ADR criteria.

## 12. Retention and Privacy

Operational telemetry retention is policy driven. Release evidence follows Quality Strategy. High-cardinality traces/metrics are bounded. Personal data is minimized and removed according to retention rules.

## 13. Synthetic and Business Checks

Synthetic checks cover health, storefront browse, authentication, cart, safe checkout simulation, SSE, and provider adapters.

Business checks detect orders without reservations, shipments without completed packing, refunds above capture, projection mismatch, or outbox with no publisher.

## 14. Monitoring Failure

Alert when collectors, exporters, dashboards, or log pipelines stop reporting. Production release validation cannot complete without critical telemetry.

## 15. Acceptance Criteria

1. Critical workflows and services have measurable signals.
2. Paging alerts use runbooks and anti-flapping controls.
3. Storage, backup, retention, and object-cost forecasts have review gates.
4. Capacity targets align with performance and infrastructure designs.
5. Sensitive data is excluded.

## Change Summary

Final version closes OPS-M02 and OPS-M03 with burn-rate/anti-flapping alert guidance and explicit database, WAL, backup, retention, and object-storage forecast gates.
