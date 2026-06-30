# Functional, Integration, and End-to-End Test Plan — Terra Commerce

**Document ID:** QA-TC-003  
**Version:** 1.0.0  
**Status:** Final  
**Owner:** QA Lead  
**Last Updated:** 2026-06-30  
**Upstream:** Final FSD, State Machine, Database Design, Contract Suite, Authorization Matrix, and TDD Suite  
**Audit:** `reviews/Quality_Test_Strategy_Suite_Audit_v1.0.0.md`

## 1. Scope

This plan verifies platform administration, tenant administration, catalog, storefront, checkout, OMS, inventory, WMS, delivery, events, notifications, reports, audit, and recovery across component, integration, and end-to-end levels.

Test IDs use `TC-FUN-<DOMAIN>-###`, `TC-INT-<DOMAIN>-###`, and `TC-E2E-<JOURNEY>-###`. Each case records upstream references, preconditions, actor, data, steps, expected state/data/event/audit, automation, owner, and evidence.

## 2. Platform and Tenant Administration

Cover:

- Tenant provisioning, rollback/recoverable failure, edit, allowed/forbidden status transitions.
- Suspension/deactivation session revocation and SSE termination.
- Owner invitation, reset, replacement, and last-owner protection.
- Support-session creation, expiry, banner, actor/effective tenant, mutation controls.
- Plans, limits, downgrade behavior, usage, flags, maintenance, health, failed-job retry.
- Store/warehouse validation and inactive warehouse guards.
- User invitation, role assignment ceiling, plan limits, deactivation/reactivation.

## 3. Catalog, Customer, and Cart

Cover:

- Product/variant/category/image create, update, publish, deactivate, archive.
- Tenant SKU uniqueness and product-plan limit.
- Publish validation and historical order immutability.
- Customer registration/login/logout/reset/rate limit and address self-ownership.
- Tenant-scoped public catalog.
- Anonymous cart ownership, item mutations, claim-token rotation, and cross-tenant/customer denial.

## 4. Checkout and Payment

Required positive flow:

1. Authenticated customer owns cart.
2. Tenant, products, prices, warehouse, rate, and inventory validate.
3. Checkout idempotency is acquired.
4. Inventory is reserved atomically.
5. Order and pending payment state commit with transition/outbox.
6. External payment initializes idempotently after commit.
7. Outcome is persisted in a second transaction.
8. Confirmation, events, notifications, audit, and SSE are verified.

Negative and edge cases:

- Guest checkout denial.
- Suspended tenant/inactive warehouse.
- Stale product/price/cart/rate.
- Insufficient or concurrently consumed stock.
- Missing package data.
- Provider rejection, timeout, unknown outcome, and retry.
- Same idempotency key with same/different payload.
- Reservation compensation or expiry according to policy.

Projection checks:

- `order_extensions` payment/fulfillment/delivery projections match authoritative state-extension aggregates after success, failure, retry, cancellation, webhook updates, and reconciliation.
- Projection lag or mismatch is detected and repairable without altering authoritative history.

## 5. OMS, Cancellation, Return, and Refund

Cover:

- Order filters, details, snapshots, notes, customer/internal timeline visibility.
- Manual payment confirmation with permission/state/version/idempotency checks.
- Cancellation from all eligible states, including stoppable picking/packing compensation.
- Rejection after irreversible fulfillment/shipment.
- Return request, review, receipt, inspection, disposition, completion.
- Partial/full refund, provider failure, retry, captured/refunded amount constraints.

Dual-control cases:

- Initiator creates high-impact request.
- Same identity cannot approve.
- Authorized second identity approves or rejects.
- Approval expiry prevents execution.
- Changed underlying state/amount invalidates pending approval.
- Override requires separately authorized emergency workflow and audit.
- Timeline, audit, transition, and event records reflect each stage.

## 6. Inventory, Receiving, Picking, and Packing

Cover:

- Reserve/release/consume arithmetic and concurrency.
- Threshold/out-of-stock events.
- Manual adjustments with reasons and conditional staff limits.
- Movement bucket before/delta/after accuracy.
- Receiving accepted/damaged/rejected equations and duplicate completion.
- Picking assignment, start/pause/resume, lines, exceptions, safe cancellation, completion.
- Packing prerequisite, dimensions/weight, exception, cancellation, completion.
- State projections remain consistent with authoritative picking/packing/fulfillment aggregates.

## 7. Shipment and Biteship

Cover:

- Rate success, invalid destination, empty result, timeout.
- Shipment guards, local pending state, provider success/failure/unknown outcome.
- Reconciliation and duplicate prevention.
- Label access controls.
- Webhook authenticity, duplicate, changed-payload conflict, unknown shipment/status, out-of-order, terminal conflict.
- Atomic webhook, tracking, shipment, order projection, transition, and outbox writes.

## 8. Events, Workers, SSE, and Notifications

Cover:

- Outbox insertion in the business transaction.
- Publisher crash after transport publish and before database acknowledgement.
- Durable consumer receipt deduplication.
- Aggregate-version gaps and reconciliation.
- Notification recipient/channel deduplication and email failure isolation.
- SSE audience projection, replay, refresh-required, revocation, suspension, shutdown, and slow consumer.
- No raw internal payload, internal note, secret, or excessive PII exposure.

## 9. Reports and Audit

Cover:

- Tenant dashboard data.
- Async report lifecycle and streaming generation.
- Query/row/resource limits.
- Current authorization revalidation at download.
- Link expiry and export audit.
- Mandatory audit action completeness, sanitization, real/effective actor, correlation, and append-only access.

## 10. Required E2E Journeys

1. Tenant onboarding through owner configuration.
2. Product publication through customer purchase and payment.
3. Payment to picking, packing, shipment, webhook delivery, and customer update.
4. Pre-fulfillment cancellation with inventory release and refund where needed.
5. Cancellation during picking with safe compensation.
6. Delivered-order return and refund with dual control when threshold applies.
7. Receiving followed by exact-stock concurrent checkout.
8. Tenant suspension blocking mutations and closing sessions/SSE.
9. Webhook duplicate/out-of-order/conflict reconciliation.
10. Outbox/worker failure and recovery without duplicate effects.

## 11. Integration Fixtures

Use current PostgreSQL migrations, Redis Streams, Medusa fixtures, Biteship/payment simulators, object/email test adapters, deterministic clock, and multiple overlapping tenants/customers.

## 12. Exit Criteria

- All critical journeys pass.
- Required domain scenarios and projection-consistency checks pass.
- Dual-control flows pass.
- No unresolved Critical/High functional defect.
- Evidence is attached to traceability rows.

## Change Summary

Final version closes QA-M03 by adding complete dual-control workflow tests and authoritative-state versus read-projection reconciliation tests.
