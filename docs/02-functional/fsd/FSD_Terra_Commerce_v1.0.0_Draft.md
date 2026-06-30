# Functional Specification Document — Terra Commerce

**Document ID:** FSD-TC-001  
**Version:** 1.0.0  
**Status:** Draft  
**Owner:** Product Owner and Technical Lead  
**Last Updated:** 2026-06-30  
**Upstream Document:** `docs/01-product/prd/PRD_Terra_Commerce_v1.0.0_Final.md`  
**Related Governance:** `docs/00-governance/`

---

## 1. Purpose

This document translates the approved Terra Commerce PRD into detailed, testable functional behavior for the MVP.

The system is a multitenant SaaS e-commerce platform using a Go gateway, Medusa.js v2, PostgreSQL, Redis Streams, Server-Sent Events, and Biteship.

MVP tenancy is fixed as:

- One tenant equals one business.
- One tenant has one store.
- One tenant has one warehouse.
- Tenant-owned data must be logically isolated.

---

## 2. Scope

This FSD covers:

- Global superadmin.
- Tenant onboarding and lifecycle.
- Tenant users, roles, and permissions.
- Store and tenant configuration.
- Catalog and product management.
- Customer account and storefront behavior.
- Cart and checkout.
- Order management.
- Inventory and warehouse workflows.
- Biteship rates, shipment creation, and tracking.
- SSE notifications.
- Audit logs.
- Reporting.
- Failure, retry, and exception handling.

Excluded from MVP:

- Multiple stores or warehouses per tenant.
- Marketplace sellers.
- Native mobile applications.
- Multi-region deployment.
- Full subscription billing automation.
- Advanced warehouse optimization.

---

## 3. Actors

| Actor | Description |
|---|---|
| Global Superadmin | Platform owner or authorized platform operator |
| Tenant Owner | Highest-authority user within a tenant |
| Tenant Administrator | General tenant administrator |
| Order Manager | Handles order and payment operations |
| Warehouse Manager | Oversees warehouse and inventory operations |
| Warehouse Staff | Performs receiving, picking, packing, and stock tasks |
| Customer Support | Assists customers and reviews order context |
| Finance Viewer | Read-only access to sales and payment reports |
| Read-Only Auditor | Read-only access to configured audit and operational records |
| Customer | Storefront user who browses, purchases, and tracks orders |
| System Worker | Background processor for retries, webhooks, events, and reports |
| Biteship | External shipping and delivery provider |

---

## 4. Global Functional Rules

### BR-001 Tenant Scope

Every tenant-owned operation must execute within a validated tenant context.

### BR-002 Server-Side Enforcement

Tenant isolation and permission checks must be enforced server-side. UI visibility alone is not sufficient.

### BR-003 Shared Infrastructure

Tenants share infrastructure but must not share business data unless explicitly allowed by a future approved requirement.

### BR-004 Auditability

Security-sensitive and business-critical operations must produce audit records.

### BR-005 Idempotency

Checkout, payment callbacks, Biteship webhooks, shipment creation, inventory deduction, refunds, and retryable background jobs must be idempotent.

### BR-006 Source of Truth

Database state and synchronous APIs are authoritative. SSE only informs clients that state may have changed.

### BR-007 Time and Currency

Tenant-facing dates must be displayed using the tenant timezone. Monetary values must use the tenant currency configuration.

### BR-008 Suspended Tenant

A suspended tenant cannot perform normal commerce operations. Read-only support access may remain available to authorized superadmins.

---

## 5. Functional Module Summary

| Module | Primary Actors | Main Outcome |
|---|---|---|
| Tenant Management | Global Superadmin | Provision and control tenants |
| Tenant Administration | Tenant Owner, Tenant Administrator | Configure tenant and users |
| Catalog | Tenant Admin roles | Manage products and variants |
| Storefront | Customer | Browse and purchase products |
| OMS | Order Manager | Process orders and payment state |
| WMS | Warehouse roles | Manage inventory, picking, and packing |
| Delivery | Order Manager, Worker, Biteship | Quote, create, and track shipments |
| Real-Time | All authenticated actors | Receive relevant live updates |
| Reporting | Tenant and global roles | Review business and operational metrics |
| Audit | Authorized roles | Trace sensitive actions |

---

# 6. Tenant Lifecycle

## FSD-FR-001 Create Tenant

**Actor:** Global Superadmin  
**Preconditions:** Superadmin is authenticated and has `tenant.create`.  
**Trigger:** Superadmin submits tenant creation form.

### Main Flow

1. Superadmin enters tenant name, slug, owner identity, plan, timezone, currency, and initial status.
2. System validates unique tenant slug.
3. System creates tenant record.
4. System creates one store.
5. System creates one warehouse placeholder.
6. System creates or invites the tenant owner.
7. System creates default tenant roles.
8. System records an audit event.
9. System returns tenant details.

### Failure Flows

- Duplicate slug: reject with validation error.
- Invalid owner email: reject.
- Partial provisioning failure: rollback or mark provisioning as failed for safe retry.

### Events

- `tenant.created`
- `tenant.owner_invited`

### Acceptance Criteria

- AC-001 Tenant, store, warehouse, and owner membership are created atomically or recoverably.
- AC-002 No duplicate tenant slug is allowed.
- AC-003 Audit entry identifies the superadmin actor.

## FSD-FR-002 Update Tenant Status

Supported transitions are Draft, Trial, Active, Past Due, Suspended, Deactivated, and Archived.

### Rules

- Only authorized global users may change tenant status.
- Suspending or deactivating a tenant invalidates active tenant sessions and SSE streams according to security policy.
- Archival requires the tenant to be deactivated first unless an emergency override is approved.

### Events

- `tenant.status_changed`
- `tenant.suspended`
- `tenant.reactivated`

## FSD-FR-003 Support Mode

Superadmin may enter audited tenant support mode.

### Requirements

- Explicit confirmation is required.
- The UI must display a visible support-mode banner.
- Support mode expires automatically.
- Passwords are never exposed.
- Every support-mode action retains both the superadmin actor and effective tenant context.

---

# 7. Tenant Administration

## FSD-FR-010 Configure Tenant

Tenant Owner or authorized administrator can manage store identity, branding, contact information, address, timezone, currency, order settings, inventory settings, delivery settings, and notification settings.

### Rules

- Currency changes after orders exist require explicit confirmation and do not modify historical order currency.
- Warehouse origin must be complete before Biteship rate requests are enabled.
- Changes to critical delivery settings are audited.

## FSD-FR-011 Manage Tenant Users

Tenant Owner can invite, deactivate, reactivate, and assign roles to users in the same tenant.

### Main Flow

1. Owner enters user identity and roles.
2. System validates tenant user limit.
3. System creates pending invitation.
4. User receives invitation.
5. User accepts and creates credentials.
6. Membership becomes active.

### Rules

- Tenant users cannot be assigned global roles.
- A tenant must retain at least one active Tenant Owner.
- A user may belong to multiple tenants in a future version, but every session must carry an explicit tenant context.

## FSD-FR-012 Role and Permission Management

Initial roles are seeded. MVP may allow role assignment but not full tenant-created custom roles unless separately approved.

Permissions include product, order, inventory, warehouse, shipment, reporting, tenant settings, and user-management actions.

---

# 8. Catalog Management

## FSD-FR-020 Create Product

**Actors:** Tenant Owner, Tenant Administrator, authorized catalog user.

### Required Data

- Product title.
- Description.
- Status.
- Category.
- At least one variant.
- SKU per variant.
- Price per variant.
- Inventory tracking setting.
- Weight and dimensions for shippable products.

### Rules

- SKU must be unique within the tenant.
- Product cannot be published if a shippable variant lacks weight or dimensions.
- Archived products are not visible to customers.

### Events

- `product.created`
- `product.updated`
- `product.published`
- `product.archived`

## FSD-FR-021 Manage Product Inventory

Inventory changes must create inventory movement records. Direct overwriting of stock without movement history is prohibited.

## FSD-FR-022 Product Search and Listing

Customers see only active products owned by the resolved tenant. Search must not return products from other tenants.

---

# 9. Customer Account and Storefront

## FSD-FR-030 Customer Registration

Customer registers within a tenant storefront using email or another approved identity method.

### Rules

- Customer identity is tenant scoped for MVP.
- Duplicate customer email behavior must follow the later authentication ADR.
- Registration, login, and password reset are rate limited.

## FSD-FR-031 Customer Login

Authenticated customer sessions are restricted to the resolved tenant and customer identity.

## FSD-FR-032 Customer Address Management

Customers can create, update, delete, and select their own addresses. Orders retain address snapshots and must not change when a customer later edits an address.

## FSD-FR-033 Product Browsing

Customers can list, search, and view active products, variants, prices, availability, images, and descriptions.

---

# 10. Cart and Checkout

## FSD-FR-040 Create and Update Cart

Customers or guests may create a cart, add items, update quantities, and remove items.

### Rules

- Cart belongs to one tenant only.
- Product and price data are revalidated during checkout.
- Quantity cannot exceed available inventory where overselling is disabled.
- Cart must not accept archived or inactive variants.

## FSD-FR-041 Calculate Shipping Rates

### Preconditions

- Cart contains shippable items.
- Customer destination is valid.
- Tenant warehouse origin is configured.
- Product dimensions and weights are complete.

### Main Flow

1. System calculates aggregate package assumptions.
2. Gateway or commerce integration requests Biteship rates.
3. System normalizes provider response.
4. Customer receives courier, service, ETA, and price.
5. Customer selects one rate.

### Failure Flow

- Biteship unavailable: display retryable shipping-rate error.
- Invalid address: request correction.
- Missing package data: block checkout and raise tenant operational alert.

## FSD-FR-042 Submit Checkout

### Main Flow

1. Customer confirms cart, address, shipping rate, and payment method.
2. System validates idempotency key.
3. System revalidates product status, price, and inventory.
4. System reserves inventory transactionally.
5. System creates order.
6. System creates payment session or pending manual payment state.
7. System returns order confirmation.
8. System emits events and SSE notifications.

### Failure Flow

- Insufficient inventory: no order is finalized; customer receives affected items.
- Payment initialization failure: order remains in a recoverable payment state or checkout rolls back according to payment design.
- Duplicate idempotency key: return the original result.

### Acceptance Criteria

- AC-040 No cross-tenant product can be added or ordered.
- AC-041 Inventory reservation and order creation are consistent.
- AC-042 Duplicate checkout requests do not create duplicate orders.

---

# 11. Order Management System

## FSD-FR-050 View Orders

Authorized tenant users can filter orders by date, order number, customer, payment status, fulfillment status, delivery status, and business status.

## FSD-FR-051 View Order Detail

Order detail includes customer and address snapshots, items, pricing, discount, shipping, payment, fulfillment, shipment, internal notes, and timeline.

## FSD-FR-052 Confirm Manual Payment

### Rules

- Only users with `order.payment_confirm` may confirm manual payment.
- Confirmation is idempotent.
- Previous and new state are audited.
- Confirmation emits `payment.confirmed` and may move the order to Ready for Fulfillment.

## FSD-FR-053 Cancel Order

### Preconditions

Cancellation eligibility depends on payment, fulfillment, and shipment state.

### Main Flow

1. Authorized actor requests cancellation with reason.
2. System validates state.
3. System cancels open fulfillment tasks where safe.
4. System releases reserved inventory where applicable.
5. System initiates refund if required.
6. System updates order status.
7. System writes audit and timeline entries.
8. System publishes events.

## FSD-FR-054 Internal Notes

Authorized tenant users may add internal notes. Customers must not see internal notes.

## FSD-FR-055 Return and Refund Request

MVP supports recording return requests and initiating approved refund workflows. Detailed eligibility and provider behavior must be finalized in the state-machine and payment specifications.

---

# 12. Inventory Management

## FSD-FR-060 Inventory Quantities

The system tracks:

- Physical stock.
- Reserved stock.
- Available stock.
- Incoming stock.
- Damaged stock.

`available_stock = physical_stock - reserved_stock`

## FSD-FR-061 Reserve Inventory

Inventory reservation occurs during checkout or another approved order transition.

### Rules

- Reservation must be atomic.
- Reservation cannot make available stock negative unless an approved overselling policy exists.
- Reservation is linked to an order.
- Expired or cancelled reservations are released idempotently.

## FSD-FR-062 Adjust Inventory

Authorized warehouse users can add or subtract stock with mandatory reason.

### Required Audit Data

- SKU.
- Before quantity.
- Changed quantity.
- After quantity.
- Reason.
- Actor.
- Timestamp.

## FSD-FR-063 Low Stock Alert

When available stock reaches or falls below threshold, system emits `inventory.low_stock` and notifies configured tenant roles.

---

# 13. Warehouse Management System

## FSD-FR-070 Create Picking Task

A picking task is created when an order becomes Ready for Fulfillment.

### Main Flow

1. Warehouse user opens fulfillment queue.
2. User starts task.
3. System assigns or records picker.
4. User confirms each SKU and quantity.
5. User completes task.
6. Order moves to Packing.

### Exceptions

- Missing stock.
- Damaged stock.
- Wrong SKU.
- Partial quantity.

Any exception pauses completion and notifies Order Management.

## FSD-FR-071 Pack Order

### Main Flow

1. Packer opens completed picking task.
2. Packer verifies items.
3. Packer enters final package weight and dimensions.
4. System validates package data.
5. Packer completes packing.
6. Order moves to Ready for Shipment.

MVP supports one package per order.

## FSD-FR-072 Warehouse Exception Resolution

Authorized user chooses one of:

- Correct inventory and continue.
- Remove unavailable item where policy allows.
- Partially fulfill where approved.
- Cancel order.
- Escalate for manual review.

Every resolution is audited.

---

# 14. Biteship Delivery

## FSD-FR-080 Create Shipment

### Preconditions

- Payment confirmed.
- Picking complete.
- Packing complete.
- Final package data present.
- Valid selected courier service.

### Main Flow

1. Authorized user or worker submits shipment creation.
2. System validates idempotency.
3. System sends normalized request to Biteship.
4. System stores provider order ID, courier, service, tracking number, price, and label URL where available.
5. Order and shipment status update.
6. System emits `shipment.created`.

### Failure Flow

- Provider timeout: mark operation retryable.
- Invalid request: mark action failed and require correction.
- Duplicate request: return existing shipment.

## FSD-FR-081 Process Biteship Webhook

### Main Flow

1. System receives webhook.
2. System verifies authenticity where supported.
3. System checks provider event idempotency.
4. System resolves tenant and shipment.
5. System persists raw reference and normalized status.
6. System updates shipment and order where applicable.
7. System publishes Redis Stream event.
8. SSE and notifications are delivered.

### Rules

- Unknown shipment events are quarantined for review.
- Duplicate events do not duplicate side effects.
- Out-of-order events are handled according to the shipment state machine.

## FSD-FR-082 Track Shipment

Tenant users and customers can view normalized shipment status and tracking history. Customers only see their own orders.

---

# 15. Server-Sent Events

## FSD-FR-090 Open SSE Connection

Authenticated clients connect through the Go gateway.

### Rules

- Connection is tenant scoped.
- Customer connection is also customer scoped.
- Permission filtering applies to event categories.
- Suspended tenant connections are rejected or terminated.
- Heartbeats keep connections alive.

## FSD-FR-091 Reconnect

Clients may send `Last-Event-ID`. The server replays supported recent events or instructs the client to perform a full refresh when replay is unavailable.

## FSD-FR-092 Event Audience

Events must exclude sensitive fields not required by the audience. Superadmin, tenant staff, and customer payloads may differ for the same underlying business change.

---

# 16. Notifications

## FSD-FR-100 In-App Notifications

In-app notifications are delivered through persistent notification records and SSE refresh events.

## FSD-FR-101 Email Notifications

Email may be sent for invitations, order confirmation, payment result, shipment creation, pickup, delivery result, and tenant suspension.

### Rules

- Notification failures do not roll back completed business transactions.
- Failed notifications are retryable.
- Templates are tenant aware where branding is supported.

---

# 17. Reporting

## FSD-FR-110 Tenant Dashboard Metrics

Metrics include sales, orders by status, fulfillment queue, low stock, shipment status, and operational alerts.

## FSD-FR-111 Tenant Reports

Tenant can request sales, order, top product, inventory movement, fulfillment, delivery, and cancellation reports.

## FSD-FR-112 Global Reports

Superadmin can review tenant growth, active tenants, order volume, plan usage, integration failures, storage, and active users.

### Rules

- Heavy reports execute asynchronously.
- Reports are tenant scoped unless explicitly global.
- Generated files require expiring authorized access.

---

# 18. Audit Logging

## FSD-FR-120 Record Audit Event

Audit records are required for:

- Tenant lifecycle changes.
- Support-mode access.
- User and role changes.
- Inventory adjustments.
- Manual payment confirmation.
- Cancellation.
- Refund action.
- Shipment creation retry or override.
- Critical global settings.

Audit record includes actor, effective tenant, action, resource, previous and new values where applicable, correlation ID, IP, user agent, and timestamp.

Audit records must be append-only from normal application flows.

---

# 19. Error and Retry Behavior

## FSD-FR-130 Standard Errors

APIs must return normalized error categories:

- Validation error.
- Authentication error.
- Authorization error.
- Not found.
- Conflict.
- Invalid state transition.
- Rate limit exceeded.
- External provider unavailable.
- Internal error.

## FSD-FR-131 Retryable Jobs

Retryable jobs use bounded retries and backoff. Permanent failure is visible to authorized operational users.

## FSD-FR-132 Dead-Letter Handling

Events or jobs exceeding retry policy are retained for investigation and manual retry according to operations policy.

---

# 20. Security Requirements

## FSD-FR-140 Authentication

System supports login, logout, password reset, expiration, and secure token or session refresh behavior.

## FSD-FR-141 Authorization

Every protected operation validates:

- Identity.
- User status.
- Tenant membership.
- Tenant status.
- Permission.
- Resource ownership.
- Current business state.

## FSD-FR-142 Rate Limiting

Authentication, password reset, checkout, shipping rate, webhook, and other abuse-sensitive endpoints are rate limited.

## FSD-FR-143 Sensitive Data

Secrets, credentials, payment-sensitive data, and unnecessary personal information must not appear in logs or SSE payloads.

---

# 21. Non-Functional Functionalization

## FSD-NFR-001 Performance

The implementation must support the PRD response and SSE targets under the agreed MVP load profile.

## FSD-NFR-002 Resource Constraint

Application processes must be configured to remain viable on 2 vCPU and 4 GB RAM, including controlled worker concurrency and bounded caches.

## FSD-NFR-003 Availability

MVP targets 99.5% monthly availability excluding approved maintenance.

## FSD-NFR-004 Observability

Requests, jobs, events, provider calls, and SSE connections must be observable through structured logs and metrics.

## FSD-NFR-005 Backup

Database backups run daily and restore procedures must be tested before production acceptance.

---

# 22. Requirement Traceability Summary

| FSD Area | PRD Area |
|---|---|
| Tenant lifecycle | Global Superadmin, Multitenancy |
| Tenant administration | Tenant Admin Dashboard |
| Catalog | Catalog Management |
| Storefront and checkout | Customer Storefront, Core Business Flows |
| OMS | Order Management System |
| Inventory and WMS | Warehouse Management System |
| Biteship | Delivery Integration |
| SSE | Real-Time Updates |
| Security and audit | Authentication, Authorization, Audit |
| Reporting | Reporting |
| Errors and retries | Reliability, Idempotency, Risks |

---

# 23. Open Decisions

The following remain unresolved and must be recorded through ADRs or dependent specifications:

1. Frontend framework.
2. Authentication implementation.
3. Payment provider.
4. Biteship credential ownership model.
5. Tenant URL model.
6. Guest checkout policy.
7. Inventory reservation expiration.
8. Order number format.
9. Tax behavior.
10. Return and refund eligibility.
11. Object storage provider.
12. Email provider.
13. Subscription billing implementation.

---

# 24. Required Downstream Documents

Before implementation begins, this FSD requires:

1. Baseline ADRs.
2. System Architecture.
3. State Machine Specification.
4. Database Design and Data Dictionary.
5. OpenAPI Contract.
6. Event Contract.
7. SSE Contract.
8. Biteship Webhook and Integration Contract.
9. Authorization Matrix.
10. Technical Design.
11. Test Strategy and requirement traceability.

---

# 25. Draft Review Checklist

- [ ] Product Owner confirms flows and scope.
- [ ] Technical Lead confirms feasibility and boundaries.
- [ ] Every PRD feature maps to an FSD section.
- [ ] Roles and permissions are consistent.
- [ ] Failure flows are sufficient.
- [ ] Tenant isolation is explicit.
- [ ] State-changing operations are ready for state-machine design.
- [ ] Open decisions are assigned to ADRs.
- [ ] Acceptance criteria are sufficiently testable.

---

## Change Summary

Initial FSD v1.0.0 draft derived from PRD v1.0.0 Final and governance v1.0.0.
