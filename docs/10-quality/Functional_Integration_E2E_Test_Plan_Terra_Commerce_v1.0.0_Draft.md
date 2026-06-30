# Functional, Integration, and End-to-End Test Plan — Terra Commerce

**Document ID:** QA-TC-003  
**Version:** 1.0.0  
**Status:** Draft  
**Owner:** QA Lead  
**Last Updated:** 2026-06-30  
**Upstream:** FSD v1.0.1, Final State Machine, Database Design, Contract Suite, Authorization Matrix, and TDD Suite

## 1. Scope

This plan verifies user-visible functionality and cross-component integration for platform administration, tenant administration, catalog, storefront, checkout, OMS, inventory, WMS, Biteship delivery, events, notifications, reporting, audit, and operational retry paths.

## 2. Test Case Convention

IDs use:

- `TC-FUN-<DOMAIN>-###`
- `TC-INT-<DOMAIN>-###`
- `TC-E2E-<JOURNEY>-###`

Each case records requirement IDs, preconditions, actors, data, steps, expected state/data/events/audit, automation level, owner, and evidence.

## 3. Platform Administration

Test scenarios:

- Create tenant with owner, plan, store, warehouse placeholder, roles, and audit.
- Reject duplicate slug and invalid owner data.
- Edit tenant profile without modifying historical order currency.
- Exercise every allowed and forbidden tenant status transition.
- Verify suspension revokes sessions and terminates SSE.
- Reset owner access, resend/revoke invitation, force reset, and replace owner while retaining one active owner.
- Start/expire support session with visible and audited effective context.
- Plan create/update/archive and assignment.
- Plan downgrade above limits: existing data readable, new creation blocked.
- Feature flag and maintenance-mode changes.
- Health, failed-job inspection, and operational retry.

## 4. Tenant Administration

- Store and warehouse configuration validation.
- Warehouse inactive blocks rates and fulfillment.
- Invite, accept, resend, revoke, deactivate, reactivate tenant users.
- Enforce user plan limits.
- Prevent last-owner deactivation.
- Assign seeded roles and reject global/over-ceiling roles.
- Verify session/authorization cache invalidation after role changes.

## 5. Catalog

- Create product with valid variant, price, dimensions, SKU, category, and media.
- Tenant-scoped SKU uniqueness.
- Product plan limit.
- Publish validation failures for missing price, dimensions, variant, or required metadata.
- Update product without modifying historical order snapshots.
- Deactivate/archive and hide from new storefront purchases.
- Variant/category/image operations.
- Tenant isolation for products and media references.

## 6. Customer and Storefront

- Registration, login, logout, password reset, throttling, and session revocation.
- Customer identity uniqueness according to approved policy.
- Address CRUD self-only.
- Public product search/list/detail only for resolved tenant and active products.
- Anonymous cart create/update/remove.
- Claim cart after authentication and invalidate claim token.
- Prevent cart claim across tenants/customers.

## 7. Checkout

Positive flow:

1. Authenticated customer owns cart.
2. Products, prices, inventory, destination, warehouse, and rate are valid.
3. Checkout idempotency acquired.
4. Inventory reserved atomically.
5. Order and payment-pending state committed.
6. External payment initialized idempotently.
7. Order confirmation returned.
8. Events, notifications, audit, and SSE projections produced.

Negative/edge cases:

- Guest attempts checkout.
- Inactive tenant or warehouse.
- Product deactivated after cart creation.
- Price changes before checkout.
- Insufficient stock.
- Missing package dimensions.
- Rate provider timeout.
- Duplicate idempotency key with same/different payload.
- Payment initialization success, rejection, timeout, and retry.
- Concurrent checkout for exact remaining stock.
- Reservation compensation/expiration according to policy.

## 8. OMS and Payment

- Order list filters and tenant isolation.
- Order detail snapshots, notes, timeline visibility.
- Manual payment confirmation positive/forbidden/stale-state cases.
- Cancellation request from every eligible state.
- Cancellation during stoppable picking/packing with compensation.
- Reject cancellation after irreversible shipment state.
- Return request, approval, rejection, receipt, inspection, completion.
- Refund approval, dual control threshold, partial/full completion, failure, and retry.
- Ensure refunded total never exceeds captured total.

## 9. Inventory and WMS

Inventory:

- Reserve/release/consume arithmetic.
- Low/out-of-stock events.
- Manual adjustments with reason and threshold permissions.
- Movement ledger before/delta/after accuracy.

Receiving:

- Accepted, damaged, rejected, mixed quantities.
- Incoming/physical/damaged bucket equations.
- Duplicate completion idempotency.

Picking:

- Create, assign, start, pause, resume, confirm lines, complete.
- Missing/damaged/wrong/partial exception handling.
- Task assignment and manager override.

Packing:

- Start and complete after picking only.
- Require positive package dimensions/weight.
- Package mismatch exception.
- Cancellation compensation.

## 10. Shipment and Biteship Integration

- Shipping-rate adapter success, empty, invalid destination, timeout.
- Shipment create after all guards.
- Timeout/unknown provider outcome followed by reconciliation.
- Prevent duplicate shipment creation.
- Label retrieval authorization.
- Webhook valid, duplicate, changed-payload conflict, invalid authentication, unknown shipment, unknown status, out-of-order event, terminal conflict.
- Verify atomic tracking, shipment, order projection, transition, and outbox writes.

## 11. Events, Workers, SSE, and Notifications

- Outbox entry in same business transaction.
- Crash after Redis publish/before outbox acknowledgement.
- Consumer duplicate receipt and idempotency.
- Aggregate version gap and reconciliation.
- Notification recipient resolution and channel deduplication.
- Email failure does not roll back business transaction.
- SSE staff/customer projection, replay, refresh-required, suspension, role revocation, slow consumer.
- No raw internal payload or sensitive fields exposed.

## 12. Reports and Audit

- Tenant-scoped dashboard metrics.
- Async report request, running, complete, failure.
- Streaming generation and row limits.
- Revalidate permission and ownership at download.
- Expired report link.
- Audit creation for all mandatory actions.
- Audit and transition records immutable through normal APIs/roles.

## 13. End-to-End Journeys

Required automated or release E2E journeys:

1. Superadmin creates tenant; owner accepts invitation and configures store/warehouse.
2. Tenant creates/publishes product; customer browses, claims cart, checks out, and pays.
3. Staff verifies payment, picks, packs, creates shipment; webhook progresses to delivered; customer receives updates.
4. Customer cancellation before fulfillment and inventory release/refund.
5. Cancellation during picking with safe stop and compensation.
6. Delivered order return and refund.
7. Receiving stock then selling exact available quantity under concurrency.
8. Tenant suspension terminates sessions/SSE and blocks mutations.
9. Biteship duplicate/out-of-order/conflict reconciliation.
10. Failed outbox/worker processing recovered without duplicate side effects.

## 14. Integration Fixtures

- PostgreSQL with current migrations.
- Redis Streams and consumer groups.
- Deterministic Medusa fixture layer.
- Biteship simulator based on normalized contract.
- Payment simulator until provider ADR is final.
- Object-storage and email test adapters.
- Controlled clock for expiry and retry tests.

## 15. Regression Selection

Changes to state machines, permissions, schemas, contracts, tenant repositories, checkout, payments, inventory, shipments, or events trigger full critical regression. Lower-risk presentation changes may use targeted suites plus smoke.

## 16. Exit Criteria

- All critical E2E journeys pass.
- Mandatory domain scenarios pass.
- No Critical/High unresolved functional defect.
- Contract, state, data, and audit assertions pass.
- Evidence links are recorded in the traceability matrix.

## Change Summary

Initial detailed functional, integration, and E2E test plan synchronized with the finalized functional and technical baseline.
