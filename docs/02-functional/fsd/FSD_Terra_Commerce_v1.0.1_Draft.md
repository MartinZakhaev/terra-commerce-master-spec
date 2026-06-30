# Functional Specification Document — Terra Commerce

**Document ID:** FSD-TC-001  
**Version:** 1.0.1  
**Status:** Draft  
**Owner:** Product Owner and Technical Lead  
**Last Updated:** 2026-06-30  
**Supersedes Draft:** `FSD_Terra_Commerce_v1.0.0_Draft.md`  
**Upstream Document:** `docs/01-product/prd/PRD_Terra_Commerce_v1.0.0_Final.md`  
**Audit Addressed:** `docs/02-functional/reviews/FSD_PRD_Consistency_Review_v1.0.0.md`

---

## 1. Purpose

This document defines detailed, testable MVP behavior for Terra Commerce, a multitenant SaaS e-commerce platform using a Go gateway, Medusa.js v2, PostgreSQL, Redis Streams, Server-Sent Events, and Biteship.

MVP tenancy is locked as one tenant, one store, and one warehouse. Tenant-owned data must remain logically isolated across APIs, database access, cache, files, events, SSE, and background jobs.

---

## 2. Scope

### 2.1 Included

- Global superadmin dashboard.
- Tenant lifecycle, plans, usage limits, configuration, monitoring, and support mode.
- Tenant settings, warehouse configuration, users, roles, and permissions.
- Catalog, variants, images, categories, publishing, and inventory.
- Customer account, storefront, cart, authenticated checkout, order confirmation, history, and tracking.
- OMS, payment verification, cancellation, return, refund, timeline, and notes.
- WMS receiving, inventory movements, picking, packing, printing, and exceptions.
- Biteship rates, shipment creation, labels, webhooks, and tracking.
- SSE, in-app and email notifications.
- Reporting, auditing, retries, dead-letter handling, and observability.

### 2.2 Excluded

- Multiple stores or warehouses per tenant.
- Marketplace or multi-seller commerce.
- Native mobile applications.
- International shipping and customs.
- Supplier and procurement management.
- Advanced warehouse slotting or route optimization.
- Enterprise BI.
- Automated custom-domain provisioning.
- Cross-tenant catalog or inventory sharing.
- Multi-region deployment.
- Offline WMS operation.
- Full automated subscription billing.
- Separate OMS and WMS backend microservices.

---

## 3. Actors

| Actor | Description |
|---|---|
| Global Superadmin | Platform owner or authorized global operator |
| Tenant Owner | Highest-authority tenant user |
| Tenant Administrator | General tenant administrator |
| Order Manager | Manages orders, payment review, cancellation, return, and refund |
| Warehouse Manager | Oversees warehouse, inventory, receiving, picking, and packing |
| Warehouse Staff | Executes receiving, picking, packing, and adjustments |
| Customer Support | Reviews customer and order context within granted permissions |
| Finance Viewer | Read-only sales and payment reporting |
| Read-Only Auditor | Read-only access to allowed audit and operational records |
| Customer | Storefront user |
| System Worker | Processes background jobs, retries, webhooks, reports, and events |
| Biteship | External delivery provider |

---

## 4. Global Business Rules

### BR-001 Tenant Scope
Every tenant-owned operation requires a validated tenant context.

### BR-002 Server-Side Enforcement
Tenant isolation, permissions, ownership, and state eligibility must be enforced server-side.

### BR-003 Shared Infrastructure
Tenants share infrastructure but never business data unless a future approved requirement explicitly allows it.

### BR-004 Auditability
Security-sensitive and business-critical operations must be audited.

### BR-005 Idempotency
Checkout, payment callbacks, Biteship webhooks, shipment creation, inventory deduction, refunds, and retryable jobs must be idempotent.

### BR-006 Source of Truth
Database state and synchronous APIs are authoritative. SSE is a notification transport, not the source of truth.

### BR-007 Time and Currency
Tenant-facing dates use tenant timezone. Historical order currency and monetary snapshots are immutable.

### BR-008 Suspended Tenant
Suspended tenants cannot perform normal commerce operations. Authorized superadmins may retain audited read-only support access.

### BR-009 Guest Behavior
Guests may browse and create a tenant-scoped anonymous cart. MVP checkout requires customer authentication. Guest checkout remains disabled until an approved ADR explicitly enables it.

### BR-010 Historical Snapshots
Orders preserve customer, address, price, tax, discount, shipping, and currency snapshots.

---

# 5. Global Superadmin

## FSD-FR-001 Create Tenant

**Actor:** Global Superadmin  
**Permission:** `tenant.create`

### Main Flow
1. Enter tenant name, slug, owner identity, plan, timezone, currency, and initial status.
2. Validate slug uniqueness and plan availability.
3. Create tenant.
4. Create one store.
5. Create one warehouse placeholder.
6. Create or invite tenant owner.
7. Seed default roles and permissions.
8. Record audit entry.
9. Emit `tenant.created` and `tenant.owner_invited`.

### Failures
- Duplicate slug: reject.
- Invalid owner identity: reject.
- Provisioning failure: rollback or persist recoverable failed-provisioning state.

## FSD-FR-002 Edit Tenant

**Permission:** `tenant.update`

Superadmin may update display name, legal name, primary contact, timezone, default currency before commercial use, plan, and operational metadata.

Rules:
- Historical orders are unaffected by currency changes.
- Plan changes follow plan transition rules.
- Critical changes are audited.

## FSD-FR-003 Tenant Status Lifecycle

Statuses: Draft, Trial, Active, Past Due, Suspended, Deactivated, Archived.

Rules:
- Only authorized global users may transition status.
- Suspension and deactivation invalidate tenant sessions and SSE streams.
- Archived requires Deactivated unless emergency override is audited.
- Invalid transitions are rejected.

Events:
- `tenant.status_changed`
- `tenant.suspended`
- `tenant.reactivated`

## FSD-FR-004 Reset Tenant Owner Access

**Permission:** `tenant.owner_access_reset`

Capabilities:
- Resend invitation.
- Revoke pending invitation.
- Force password reset.
- Revoke active sessions.
- Replace primary owner only after ensuring at least one active owner remains.

All actions are audited and trigger a security notification to the affected owner.

## FSD-FR-005 Support Mode

Requirements:
- Explicit confirmation.
- Visible support-mode banner.
- Expiring session.
- No password exposure.
- Original superadmin actor and effective tenant context retained in every audit record.
- Support mode cannot silently bypass prohibited actions.

---

# 6. Subscription Plans and Limits

## FSD-FR-010 Manage Plans

**Actor:** Global Superadmin  
**Permissions:** `plan.read`, `plan.create`, `plan.update`, `plan.archive`

Plan fields:
- Name and code.
- Status.
- Trial duration.
- Maximum products.
- Monthly order limit.
- Tenant-user limit.
- Media-storage limit.
- SSE-connection limit.
- Enabled modules.
- Enabled reports.
- Soft or hard enforcement policy per limit.

Rules:
- Plan code is unique.
- Archived plans cannot be newly assigned.
- Existing tenant assignments remain until migrated.

## FSD-FR-011 Assign or Change Tenant Plan

Main flow:
1. Select tenant and target plan.
2. Calculate current usage against target limits.
3. Display incompatibilities.
4. Require confirmation for downgrade.
5. Apply immediately or on configured effective date.
6. Audit the change.
7. Emit `tenant.plan_changed`.

Downgrade rules:
- Existing resources above target limits remain readable.
- New resource creation is blocked when hard limits are exceeded.
- Product deletion, archival, user deactivation, and data export remain available.

## FSD-FR-012 Enforce Plan Limits

Before creating limited resources, the system checks current usage and effective plan.

Limit outcomes:
- Allowed.
- Soft warning.
- Hard rejection with normalized `plan_limit_exceeded` error.

Limit checks must occur server-side and be tenant scoped.

## FSD-FR-013 View Plan Usage

Superadmin and authorized tenant owners can view current usage, limit, remaining capacity, and exceeded status for each plan dimension.

---

# 7. Global Configuration and Operations

## FSD-FR-020 Global Settings

**Permission:** `platform.settings.update`

Manage:
- Default currency and timezone.
- Email settings.
- File-size limits.
- Default plan.
- Notification templates.
- Platform branding.
- Biteship platform configuration where applicable.

Secrets must be masked and never returned after storage.

## FSD-FR-021 Feature Flags

Superadmin may enable or disable flags globally or for selected tenants.

Rules:
- Flag changes are audited.
- Flag evaluation is server-side.
- Disabling a feature must not corrupt existing data.

## FSD-FR-022 Maintenance Mode

Superadmin may enable maintenance mode with message, start time, and optional end time.

Behavior:
- Normal mutations are blocked.
- Health checks, monitoring, and authorized support access remain available.
- Customer-facing clients receive a normalized maintenance response.

## FSD-FR-023 Platform Health Dashboard

Display:
- CPU, memory, and disk usage.
- PostgreSQL and Redis health.
- Active SSE connections.
- Failed and delayed jobs.
- Biteship failure counts.
- API error rate and latency summary.
- Recent critical alerts.

## FSD-FR-024 Operational Failure Management

Authorized operators may inspect failed jobs and provider calls, view sanitized failure details, retry eligible operations, acknowledge alerts, and link incidents.

Retry actions must be idempotent and audited.

---

# 8. Tenant Administration

## FSD-FR-030 Configure Store

Manage store name, description, logo, favicon, brand colors, contact details, store address, timezone, currency, order settings, inventory settings, delivery settings, and notifications.

Rules:
- Currency changes do not alter historical orders.
- Media uploads follow tenant storage limits.
- Critical delivery changes are audited.

## FSD-FR-031 Configure Warehouse

Fields:
- Warehouse name.
- Address.
- Contact person.
- Contact phone.
- Operational status.
- Biteship origin area.
- Operating hours.

Rules:
- One warehouse per tenant in MVP.
- Complete origin data is required before shipping-rate requests.
- Inactive warehouse blocks new fulfillment and rate requests.

## FSD-FR-032 Manage Tenant Users

Tenant Owner may invite, resend invitation, revoke invitation, deactivate, reactivate, and assign roles.

Rules:
- Enforce plan user limit.
- Global roles cannot be assigned.
- At least one active Tenant Owner must remain.
- Deactivation revokes sessions.

## FSD-FR-033 Roles and Permissions

Seeded roles:
- Tenant Owner.
- Tenant Administrator.
- Order Manager.
- Warehouse Manager.
- Warehouse Staff.
- Customer Support.
- Finance Viewer.
- Read-Only Auditor.

MVP supports assignment of seeded roles. Custom role creation is deferred unless approved separately.

---

# 9. Catalog Management

## FSD-FR-040 Create Product

Required data:
- Title.
- Description.
- Status.
- Category.
- At least one variant.
- Unique tenant-scoped SKU.
- Price.
- Inventory-tracking setting.
- Weight and dimensions for shippable variants.

## FSD-FR-041 Update Product

Authorized users may update descriptive, pricing, categorization, media, shipping, and variant data.

Rules:
- Changes do not alter historical order snapshots.
- SKU changes are audited.
- Removing a variant referenced by historical orders requires archival, not destructive deletion.

## FSD-FR-042 Publish Product

A product may become Active only when:
- At least one active variant exists.
- Price is valid.
- Shippable variants have weight and dimensions.
- Required media or category policy is satisfied.

Emit `product.published`.

## FSD-FR-043 Deactivate or Archive Product

Inactive products are hidden from new purchases but remain manageable. Archived products are read-only except restoration where allowed.

Existing orders remain accessible.

## FSD-FR-044 Manage Variants

Create, update, activate, deactivate, and archive variants. SKU must be unique within tenant.

## FSD-FR-045 Manage Categories

Create, update, assign, reorder, deactivate, and archive tenant categories.

## FSD-FR-046 Manage Product Images

Upload, reorder, set primary image, and remove unused images subject to storage limits and file validation.

## FSD-FR-047 Product Search and Listing

Customers see only active products and variants belonging to the resolved tenant. Search must never cross tenant boundaries.

---

# 10. Customer and Storefront

## FSD-FR-050 Customer Registration

Customer registers within a tenant storefront. Registration, login, and password reset are rate limited.

Customer identity is tenant scoped for MVP. Duplicate email behavior is finalized by authentication ADR.

## FSD-FR-051 Customer Login and Logout

Sessions are restricted to customer identity and resolved tenant. Logout revokes or invalidates the active session.

## FSD-FR-052 Customer Addresses

Customers manage only their own addresses. Orders preserve address snapshots.

## FSD-FR-053 Browse Products

Customers can list, search, filter, and view active products, variants, prices, stock availability, images, and descriptions.

## FSD-FR-054 Order Confirmation

After successful order creation, the customer receives a confirmation page containing order number, item summary, amount, payment status, shipping selection, and next action.

## FSD-FR-055 Payment Status Display

Customers can view normalized payment status for their own orders. Sensitive provider details are hidden.

## FSD-FR-056 Customer Order History

Authenticated customers can list their own orders, filter by date and status, and open order details.

## FSD-FR-057 Customer Shipment Tracking

Customers can view normalized status and tracking history only for their own orders.

---

# 11. Cart and Checkout

## FSD-FR-060 Cart Management

Customers and anonymous guests may create a tenant-scoped cart, add, update, and remove items.

Rules:
- A cart belongs to one tenant.
- Inactive or archived variants cannot be added.
- Price and availability are revalidated at checkout.
- Guest cart may be claimed by a customer after authentication.

## FSD-FR-061 Shipping Rates

Preconditions:
- Shippable items exist.
- Destination is valid.
- Warehouse is active and configured.
- Package data is complete.

Return courier, service, ETA, and price.

Failures:
- Provider unavailable: retryable error.
- Invalid destination: validation error.
- Missing package data: checkout blocked and tenant alert created.

## FSD-FR-062 Authenticated Checkout

MVP checkout requires authenticated customer.

Flow:
1. Authenticate or claim guest cart.
2. Confirm cart, address, shipping rate, and payment method.
3. Validate idempotency key.
4. Revalidate products, prices, plan status, tenant status, and inventory.
5. Reserve inventory transactionally.
6. Create order.
7. Initialize payment or manual-payment state.
8. Return order confirmation.
9. Emit order and notification events.

Failures:
- Insufficient inventory: no finalized order.
- Payment initialization failure: recoverable payment state or rollback according to payment design.
- Duplicate idempotency key: return original outcome.

---

# 12. Order Management System

## FSD-FR-070 Order List

Authorized tenant users filter by date, order number, customer, payment status, fulfillment status, delivery status, and business status.

## FSD-FR-071 Order Detail

Includes customer and address snapshots, items, pricing, discount, tax, shipping, payment, fulfillment, shipment, internal notes, and timeline.

## FSD-FR-072 Order Timeline

Timeline entries include:
- Order created.
- Payment initiated and confirmed.
- Inventory reserved and released.
- Picking started and completed.
- Packing started and completed.
- Shipment created.
- Courier pickup and tracking changes.
- Delivery.
- Cancellation, return, and refund.
- Manual administrative actions.

Rules:
- Chronological ordering.
- Append-only from normal flows.
- Entries record actor and timestamp where applicable.
- Customer-visible and internal-only entries are explicitly classified.

## FSD-FR-073 Confirm Manual Payment

Requires `order.payment_confirm`. Confirmation is idempotent, audited, emits `payment.confirmed`, and transitions only when valid.

## FSD-FR-074 Cancel Order

Validate order, payment, fulfillment, and shipment states; cancel open tasks where safe; release inventory; initiate refund when required; audit; emit events.

## FSD-FR-075 Internal Notes

Authorized users may create internal-only notes. Customers never receive these notes.

## FSD-FR-076 Return Request

Record request reason, requested items, quantity, evidence where allowed, customer notes, and review status.

## FSD-FR-077 Refund Workflow

Authorized users may approve, reject, or initiate refund according to payment and state-machine rules. Refund actions are idempotent and audited.

---

# 13. Inventory Management

## FSD-FR-080 Inventory Quantities

Track physical, reserved, available, incoming, and damaged stock.

`available_stock = physical_stock - reserved_stock`

## FSD-FR-081 Reserve and Release Inventory

Reservations are atomic, order-linked, tenant scoped, and cannot make available stock negative unless an approved overselling policy exists. Cancellation and expiration release reservations idempotently.

## FSD-FR-082 Adjust Inventory

Authorized users add or subtract stock with mandatory reason. Record SKU, before, delta, after, actor, reason, reference, and timestamp.

## FSD-FR-083 Low and Out-of-Stock Alerts

Emit `inventory.low_stock` at or below threshold and `inventory.out_of_stock` when available stock reaches zero. Notify configured roles.

## FSD-FR-084 Inventory Movement History

Authorized users filter movements by SKU, type, actor, date, and reference.

---

# 14. Warehouse Management System

## FSD-FR-090 Receive Stock

Flow:
1. Warehouse user creates receiving record.
2. Select SKU and expected quantity where applicable.
3. Enter received, damaged, and rejected quantities.
4. Validate totals.
5. Reduce incoming stock where applicable.
6. Increase physical and damaged stock appropriately.
7. Create movement records.
8. Audit completion.

Receiving completion is idempotent and cannot be applied twice.

## FSD-FR-091 Picking Task

Created when order becomes Ready for Fulfillment.

Flow:
1. Open queue.
2. Start task and record picker.
3. Confirm each SKU and quantity.
4. Report exceptions where needed.
5. Complete task.
6. Move order to Packing.

## FSD-FR-092 Print Picking List

Authorized users may generate and print or download a tenant-branded picking list containing order reference, SKU, variant, quantity, and warehouse instructions. Internal customer data must be minimized.

## FSD-FR-093 Pack Order

Verify items, enter final package weight and dimensions, validate package data, complete packing, and move order to Ready for Shipment. MVP supports one package per order.

## FSD-FR-094 Warehouse Exceptions

Exceptions include missing stock, damaged stock, wrong SKU, and partial quantity. Resolution options: correct inventory, remove item where policy allows, partial fulfillment where approved, cancel, or manual escalation. Every resolution is audited.

## FSD-FR-095 Shipping Label

After shipment creation, authorized users may preview, download, and print Biteship label where available. Label access is tenant scoped and audited when required.

---

# 15. Biteship Delivery

## FSD-FR-100 Create Shipment

Preconditions:
- Payment confirmed.
- Picking complete.
- Packing complete.
- Final package data present.
- Valid selected courier service.

Flow:
1. Validate idempotency.
2. Send normalized request.
3. Store provider order ID, courier, service, tracking number, cost, status, and label URL.
4. Update order and shipment.
5. Emit `shipment.created`.

## FSD-FR-101 Process Webhook

Verify authenticity where supported, deduplicate provider event, resolve tenant and shipment, persist raw reference and normalized status, update order, publish Redis Stream event, then deliver SSE and notifications.

Unknown events are quarantined. Out-of-order events follow shipment state-machine rules.

## FSD-FR-102 Canonical Delivery Statuses

Canonical statuses:
- Shipment pending.
- Courier assigned.
- Awaiting pickup.
- Picked up.
- In transit.
- Out for delivery.
- Delivered.
- Delivery failed.
- Returned to sender.
- Cancelled.

Provider statuses must map to exactly one canonical status or `unknown_pending_review`.

## FSD-FR-103 Tracking

Tenant users and customers view canonical status and tracking history according to authorization.

---

# 16. Server-Sent Events

## FSD-FR-110 Open Connection

Authenticated clients connect through Go gateway. Connections are tenant scoped; customer connections are customer scoped; suspended tenants are rejected or terminated; heartbeats are sent.

## FSD-FR-111 Reconnection

Clients send `Last-Event-ID`. Server replays supported recent events or requires full refresh.

## FSD-FR-112 Event Envelope

Contains event ID, type, tenant ID, resource type, resource ID, timestamp, payload, and schema version.

## FSD-FR-113 Audience Filtering

Sensitive fields are removed according to superadmin, tenant-user, or customer audience.

---

# 17. Notifications

## FSD-FR-120 Notification Events

Required events and recipients:

| Event | Default recipients |
|---|---|
| New order | Order Manager, Tenant Owner |
| Payment confirmed or failed | Customer, Order Manager |
| Order cancelled | Customer, Order Manager |
| Low or out of stock | Warehouse Manager, Tenant Owner |
| Picking exception | Warehouse Manager, Order Manager |
| Shipment created | Customer, Order Manager |
| Courier pickup | Customer |
| Delivery completed or failed | Customer, Order Manager |
| Tenant suspended | Tenant Owner |
| User invitation | Invited user |

## FSD-FR-121 In-App Notifications

Persist notification records and publish SSE refresh events. Users may mark notifications read.

## FSD-FR-122 Email Notifications

Email failures do not roll back business transactions. Failed sends are retryable. Templates may use tenant branding.

---

# 18. Dashboards and Reports

## FSD-FR-130 Tenant Dashboard

Display:
- Sales.
- Total orders and status counts.
- Orders requiring action.
- Fulfillment queue.
- Low-stock products.
- Out-of-stock products.
- Awaiting pickup.
- Delivery updates.
- Operational alerts.

## FSD-FR-131 Tenant Reports

Sales, orders, top products, low stock, out of stock, inventory movement, fulfillment, delivery, and cancellation.

## FSD-FR-132 Global Reports

Tenant growth, active tenants, order volume by tenant, platform order volume, plan usage, integration failures, storage usage, and active users.

Heavy reports are asynchronous and generated files use expiring authorized access.

---

# 19. Audit Logging

## FSD-FR-140 Audit Event

Required for tenant lifecycle, support mode, plan changes, user and role changes, inventory receiving and adjustment, payment confirmation, cancellation, return/refund, shipment retry or override, feature flags, maintenance mode, and critical settings.

Fields: audit ID, tenant where applicable, actor, effective actor context, action, resource, before and after values where applicable, IP, user agent, correlation ID, timestamp.

Audit records are append-only from normal application flows.

---

# 20. Errors, Jobs, and Retries

## FSD-FR-150 Standard Errors

- Validation.
- Authentication.
- Authorization.
- Not found.
- Conflict.
- Invalid state transition.
- Plan limit exceeded.
- Rate limit exceeded.
- Provider unavailable.
- Maintenance mode.
- Internal error.

## FSD-FR-151 Retryable Jobs

Use bounded retries and backoff. Permanent failure is visible to authorized operators.

## FSD-FR-152 Dead-Letter Handling

Retain failed events and jobs for investigation and controlled manual retry.

---

# 21. Security

## FSD-FR-160 Authentication

Support login, logout, password reset, session expiration, and secure refresh or server-side sessions.

## FSD-FR-161 Authorization

Validate identity, user status, tenant membership, tenant status, role/permission, resource ownership, plan availability where relevant, and business state.

## FSD-FR-162 Rate Limiting

Apply to authentication, password reset, checkout, shipping rates, webhooks, report generation, and other abuse-sensitive operations.

## FSD-FR-163 Sensitive Data

Secrets, credentials, payment-sensitive data, and unnecessary personal information must not appear in logs, reports, notifications, or SSE payloads.

---

# 22. Non-Functional Requirements

## FSD-NFR-001 Infrastructure
2 vCPU, 4 GB RAM, Linux, SSD, single region.

## FSD-NFR-002 Performance
- Cached API p95 below 300 ms.
- Standard API p95 below 800 ms.
- Internal complex order operation below 2 seconds when not provider-bound.
- SSE propagation below 3 seconds.
- Dashboard load below 3 seconds on typical connection.
- Job start within 10 seconds.

## FSD-NFR-003 Capacity Assumptions
- 20 active tenants.
- 100 tenant admin users.
- 1,000 daily orders platform-wide.
- 100 concurrent storefront users.
- 50 concurrent SSE connections.
- 10,000 active products.

## FSD-NFR-004 Availability
99.5% monthly excluding approved maintenance.

## FSD-NFR-005 Observability
Structured logs and metrics for requests, jobs, events, provider calls, SSE, PostgreSQL, Redis, CPU, memory, disk, latency, and error rate.

## FSD-NFR-006 Backup and Restore
Daily database backups and tested restore before production acceptance.

## FSD-NFR-007 Repository and Release Governance
Authoritative documents remain in master-spec. Application repositories are Git submodules under `apps/`, pinned to exact commits for each release.

---

# 23. PRD Acceptance Traceability

| PRD Acceptance Criterion | FSD Coverage | Status |
|---|---|---|
| 1. Create, activate, suspend, inspect tenant | FR-001, FR-002, FR-003, FR-023 | Covered |
| 2. Manage tenant users and permissions | FR-032, FR-033 | Covered |
| 3. Configure store and warehouse | FR-030, FR-031 | Covered |
| 4. Create and publish products | FR-040 through FR-046 | Covered |
| 5. Browse, cart, rates, order | FR-053, FR-060, FR-061, FR-062 | Covered |
| 6. Safe tenant-isolated reservation | BR-001, BR-002, FR-081 | Covered |
| 7. Tenant staff process orders | FR-070 through FR-077 | Covered |
| 8. Warehouse picking and packing | FR-091 through FR-094 | Covered |
| 9. Create and track Biteship shipment | FR-100 through FR-103 | Covered |
| 10. Idempotent delivery webhooks | BR-005, FR-101 | Covered |
| 11. Authenticated tenant-scoped SSE | FR-110 through FR-113 | Covered |
| 12. Suspended tenant blocked | BR-008, FR-003, FR-110 | Covered |
| 13. Critical actions audited | FR-140 | Covered |
| 14. Cross-tenant access tests pass | BR-001, BR-002, downstream test strategy | Covered by requirement |
| 15. Load test on target server | NFR-001 through NFR-003 | Covered by requirement |
| 16. Backup and restore tested | NFR-006 | Covered |
| 17. Documents stored in master-spec | NFR-007 | Covered |
| 18. Apps linked as submodules | NFR-007 | Covered |

---

# 24. Open Decisions and ADR Targets

1. Frontend framework.
2. Authentication implementation.
3. Payment provider.
4. Biteship credential ownership.
5. Tenant URL model.
6. Inventory reservation expiration.
7. Order-number format.
8. Tax behavior.
9. Return/refund eligibility detail.
10. Object-storage provider.
11. Email provider.
12. Subscription billing implementation.
13. Customer identity uniqueness across tenants.

Guest checkout is explicitly disabled for MVP until an ADR changes the decision.

---

# 25. Required Downstream Documents

- Baseline ADRs.
- System Architecture.
- State Machines.
- Database Design and Data Dictionary.
- OpenAPI Contract.
- Event, SSE, and Webhook Contracts.
- Authorization Matrix.
- Biteship Integration Specification.
- Technical Design.
- Infrastructure Specification.
- Test Strategy and traceability.
- Security Threat Model.
- Operational Runbook.

---

# 26. Review Checklist

- [x] PRD scope represented.
- [x] High-priority audit findings addressed.
- [x] Medium-priority audit findings addressed.
- [x] Full PRD acceptance traceability added.
- [x] Guest checkout ambiguity resolved for MVP.
- [ ] Product Owner confirms functional behavior.
- [ ] Technical Lead confirms feasibility.
- [ ] ADR owners assigned.
- [ ] Architecture and state-machine review completed.

---

## Change Summary

Version 1.0.1 closes findings from `FSD_PRD_Consistency_Review_v1.0.0.md`, including plan management, global operations, inventory receiving, guest-checkout clarification, tenant-owner access reset, product lifecycle, customer post-checkout flows, warehouse configuration, timeline, printing, notification mapping, delivery statuses, non-goals, and acceptance traceability.
