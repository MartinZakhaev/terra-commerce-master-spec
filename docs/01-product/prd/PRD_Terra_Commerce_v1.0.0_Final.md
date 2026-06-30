# Product Requirements Document — Terra Commerce

**Document ID:** PRD-TC-001  
**Version:** 1.0.0  
**Status:** Final  
**Scope:** MVP  
**Owner:** CEO / Product Owner  
**Source of Truth:** `MartinZakhaev/terra-commerce-master-spec`  
**Initial Infrastructure:** 2 vCPU, 4 GB RAM  
**Core Technology:** Go Gateway, Medusa.js v2, PostgreSQL, Redis Streams, SSE, Biteship

---

## 1. Executive Summary

Terra Commerce is a multitenant SaaS e-commerce platform that enables multiple businesses, referred to as tenants, to operate independent online stores on shared infrastructure.

Each tenant receives a tenant administration dashboard, catalog management, an Order Management System (OMS), a Warehouse Management System (WMS), Biteship delivery integration, and real-time operational updates through Server-Sent Events (SSE).

The platform owner operates a global superadmin dashboard for tenant lifecycle management, plans, platform configuration, monitoring, usage, and support operations.

For MVP:

- One tenant equals one business.
- One tenant has one store.
- One tenant has one warehouse.
- Tenant data is logically isolated.
- The system uses a modular-monolith architecture.
- The system must operate within an initial 2 vCPU and 4 GB RAM server target.

---

## 2. Product Vision

Provide a lightweight, reliable, extensible SaaS commerce platform combining storefront operations, order processing, inventory control, warehouse fulfillment, and delivery integration in one tenant-isolated system.

---

## 3. Product Goals

### 3.1 Primary Goals

1. Enable the platform owner to create, manage, suspend, and monitor tenants.
2. Enable each tenant to operate an independent online store.
3. Provide tenant-scoped OMS and WMS capabilities.
4. Integrate domestic delivery through Biteship.
5. Deliver real-time UI updates through SSE.
6. Enforce tenant isolation across APIs, storage, cache, events, files, and background jobs.
7. Operate within the initial server constraint.
8. Preserve a future path to multiple stores and warehouses per tenant.
9. Maintain all authoritative specifications in the master-spec repository.

### 3.2 Secondary Goals

- Support tenant branding and basic white-label configuration.
- Provide auditable platform and tenant operations.
- Minimize infrastructure and operational cost during MVP.
- Allow future payment, logistics, notification, and analytics integrations.

---

## 4. MVP Non-Goals

The MVP excludes:

- Multiple stores or warehouses per tenant.
- Marketplace or multi-seller commerce.
- Native mobile applications.
- International shipping and customs.
- Supplier and procurement management.
- Advanced warehouse slotting and routing.
- Enterprise BI.
- Automated custom-domain provisioning.
- Cross-tenant catalog or inventory sharing.
- Multi-region deployment.
- Offline WMS operation.
- Full subscription-billing automation.
- Separate OMS and WMS backend microservices.

---

## 5. Users and Roles

### 5.1 Global Superadmin

Platform owner or authorized internal platform team. Manages tenants, plans, global settings, monitoring, support access, feature flags, and global audit records.

### 5.2 Tenant Owner

Highest-authority tenant user. Manages tenant settings, users, roles, catalog, OMS, WMS, delivery configuration, and reports.

### 5.3 Tenant Administrator

Performs tenant administration based on assigned permissions.

### 5.4 Order Management Staff

Handles order review, payment verification, cancellation, fulfillment coordination, returns, and refunds.

### 5.5 Warehouse Staff

Handles stock receiving, adjustments, picking, packing, shipment preparation, and discrepancies.

### 5.6 Customer

Browses products, manages a cart, places an order, pays, chooses delivery, and tracks shipment.

---

## 6. Product Components

1. Global superadmin dashboard.
2. Tenant admin dashboard.
3. Tenant OMS.
4. Tenant WMS.
5. Customer storefront.
6. Go API gateway.
7. Medusa.js v2 commerce engine.
8. Biteship integration.
9. SSE delivery layer.
10. Redis cache and Streams event transport.
11. Background workers.
12. PostgreSQL database.
13. Audit and observability layer.
14. Master specification repository.

---

## 7. Architecture Principles

### 7.1 Modular Monolith

The MVP uses a modular monolith. Independently deployed services may only be introduced when justified by security, scaling, operational isolation, or measured performance needs.

### 7.2 Go Gateway

The Go gateway is the public API entry point and is responsible for:

- Authentication validation.
- Tenant resolution.
- Authorization enforcement.
- Request routing and validation.
- Rate limiting.
- Correlation IDs and structured request logging.
- Response normalization.
- SSE connection handling.
- Platform-level APIs.

The gateway must not duplicate Medusa commerce logic unnecessarily.

### 7.3 Medusa.js v2

Medusa.js v2 is the core commerce engine responsible for products, variants, prices, customers, carts, orders, inventory, promotions, fulfillment, returns, and refund workflows.

Custom Medusa modules may implement tenant-aware and Terra Commerce-specific behavior.

### 7.4 PostgreSQL

The MVP uses a shared database and shared schema. Tenant-owned records must be tenant scoped directly or through enforceable ownership relationships.

Tenant isolation must be enforced by tenant-aware data access, authorization, constraints, and automated tests. PostgreSQL Row-Level Security may be evaluated later.

### 7.5 Redis

Redis is used for caching, rate-limit counters, idempotency keys, short-lived coordination, locks where required, Redis Streams, and real-time event fan-out.

### 7.6 SSE

SSE is used for server-to-client real-time updates. REST APIs remain responsible for commands and mutations. The database and APIs remain the source of truth.

### 7.7 Workers

Workers process webhooks, Biteship shipment creation, delivery synchronization, notifications, retries, reports, cleanup, and event publication.

---

## 8. Multitenancy Requirements

### 8.1 Tenant Model

Each tenant has one store and one warehouse in MVP, with multiple administrative users, customers, products, orders, inventory records, and shipments.

### 8.2 Required Tenant Data

- Tenant ID.
- Unique slug.
- Display and legal name where applicable.
- Status.
- Subscription plan.
- Store configuration.
- Warehouse configuration.
- Branding configuration.
- Created and updated timestamps.

### 8.3 Tenant Resolution

Tenant context may be resolved from subdomain, URL path, authenticated membership, or trusted internal context. Client-supplied tenant IDs must never be trusted without authorization validation.

### 8.4 Tenant Statuses

- Draft.
- Trial.
- Active.
- Past due.
- Suspended.
- Deactivated.
- Archived.

### 8.5 Isolation Rules

- Tenant users cannot access another tenant's resources.
- Cache keys, files, events, jobs, and logs must carry tenant context.
- SSE subscriptions are tenant scoped.
- Provider configuration is tenant scoped when tenant-owned credentials are supported.
- Superadmin cross-tenant access is explicit and audited.
- Automated tests must attempt and reject cross-tenant access.

---

## 9. Global Superadmin Requirements

The global superadmin dashboard must support:

- Tenant creation, editing, activation, suspension, deactivation, and archival.
- Tenant owner access reset.
- Plan assignment and changes.
- Tenant usage and operational status.
- Global settings and feature flags.
- Platform monitoring and integration-failure visibility.
- Global audit review.
- Audited tenant support-mode impersonation.

Support mode must require confirmation, show a visible indicator, have a limited duration, and never reveal passwords.

Plan limits may cover products, monthly orders, tenant users, media storage, SSE connections, modules, and reports.

---

## 10. Tenant Admin Requirements

The tenant dashboard must show sales, order status summaries, fulfillment queues, low stock, out-of-stock products, shipment updates, and operational alerts.

Tenant settings include store identity, branding, contact details, address, currency, timezone, order, inventory, delivery, and notification settings.

Tenant owners can invite, deactivate, and manage users and roles.

Initial roles:

- Tenant Owner.
- Tenant Administrator.
- Order Manager.
- Warehouse Manager.
- Warehouse Staff.
- Customer Support.
- Finance Viewer.
- Read-Only Auditor.

Authorization must be action based, for example `product.create`, `order.cancel`, `inventory.adjust`, `warehouse.pick`, `shipment.create`, and `user.manage`.

---

## 11. Catalog and Storefront

Tenant administrators can create, edit, publish, deactivate, and archive products; manage variants, SKUs, prices, categories, images, inventory, weight, and dimensions.

Every shippable product must provide weight, length, width, and height.

The storefront must support product listing and search, product details and variants, carts, registration and login, optional guest checkout, addresses, Biteship rate calculation, checkout, payment status, order confirmation, history, and tracking.

All storefront APIs pass through the Go gateway.

---

## 12. Order Management System

### 12.1 Order Data

Every order includes tenant, order number, customer, billing and shipping addresses, items, quantities, prices, discounts, shipping, taxes where applicable, total, currency, payment status, fulfillment status, delivery status, business status, timestamps, and history.

### 12.2 Business Statuses

- Draft.
- Pending payment.
- Payment pending verification.
- Paid.
- Processing.
- Ready for fulfillment.
- Picking.
- Packing.
- Ready for shipment.
- Shipped.
- Delivered.
- Completed.
- Cancellation requested.
- Cancelled.
- Return requested.
- Returned.
- Refund pending.
- Refunded.
- Failed.

A later state-machine specification must define mappings to Medusa internal states.

### 12.3 Order Operations

Authorized users can view orders, confirm manual payments, cancel eligible orders, add internal notes, begin fulfillment, print picking lists, mark items picked and packed, request rates, select courier service, create shipments, view tracking, process returns, initiate refunds, and view the order timeline.

---

## 13. Warehouse Management System

Each tenant has one warehouse with an address, contact, operating status, Biteship origin, and operating hours.

The WMS supports physical, reserved, available, incoming, and damaged stock; low-stock thresholds; inventory adjustments; movements; reservations; releases; and fulfillment deductions.

`available_stock = physical_stock - reserved_stock`

Movement types include initial stock, receiving, manual addition, manual deduction, reservation, release, fulfillment deduction, return, damage, and correction.

Each movement records tenant, warehouse, SKU, before quantity, changed quantity, after quantity, type, reference, actor, reason, and timestamp.

Picking and packing workflows must support task start, item confirmation, missing-stock reporting, final package weight and dimensions, one package per order in MVP, label printing where supported, and shipment readiness.

Stock discrepancies must create an auditable exception and notify order management. Inventory must not be silently changed.

---

## 14. Biteship Integration

Rate calculation uses warehouse origin, customer destination, package weight, package dimensions, and courier preferences.

The storefront displays courier, service, estimated delivery duration, and price.

Shipment creation requires confirmed payment, completed picking and packing, and final package data.

The platform stores Biteship order ID, courier, service, tracking number, label URL where available, shipping cost, status, tracking history, and support reference data.

Biteship webhook processing must support validation where available, idempotency, duplicate detection, event logging, retries, tenant resolution, order and shipment synchronization, and SSE publication.

Delivery statuses include shipment pending, courier assigned, awaiting pickup, picked up, in transit, out for delivery, delivered, failed, returned to sender, and cancelled.

When Biteship is unavailable, users receive clear errors, eligible operations are retryable, failures are visible, and duplicate shipments are prevented.

---

## 15. Real-Time Updates

SSE use cases include new orders, order and payment changes, inventory updates, low stock, picking, packing, shipment creation, delivery updates, system alerts, tenant suspension, and background-job results.

Logical channels:

- `tenant:{tenant_id}:orders`
- `tenant:{tenant_id}:inventory`
- `tenant:{tenant_id}:warehouse`
- `tenant:{tenant_id}:shipments`
- `tenant:{tenant_id}:alerts`
- `user:{user_id}:notifications`

Every event includes event ID, type, tenant ID, resource type, resource ID, timestamp, payload, and schema version.

SSE requires authentication and audience-based filtering. Clients must support reconnect, `Last-Event-ID`, deduplication, heartbeat, and full refresh after long disconnection.

---

## 16. Authentication, Authorization, and Audit

The platform supports login, logout, password reset, session expiration, secure refresh handling or server-side sessions, tenant-aware tokens, and a separate superadmin boundary.

Every protected action validates identity, user status, tenant membership, tenant status, role, permission, resource ownership, and business-state eligibility.

Customers can only access their own profile, addresses, carts, orders, and shipment tracking.

Critical actions must be audited with tenant, actor, action, resource, before and after values where relevant, IP address, user agent, correlation ID, and timestamp.

---

## 17. Notifications and Reporting

MVP notification channels are in-app and email.

Events include new order, payment result, cancellation, low stock, picking exception, shipment creation, pickup, delivery result, suspension, and user invitation.

Tenant reports include sales, orders by status, top products, low stock, inventory movement, fulfillment, delivery, and cancellation.

Superadmin reports include tenant growth, active tenants, platform order volume, plan usage, integration failures, storage, and active users.

Heavy reports must run asynchronously.

---

## 18. Non-Functional Requirements

### 18.1 Infrastructure

- 2 vCPU.
- 4 GB RAM.
- Linux.
- SSD storage.
- Single region.

Planning memory targets:

- Go gateway: 100–250 MB.
- Medusa: 500–900 MB.
- Worker: 250–600 MB.
- PostgreSQL: 700–1,200 MB.
- Redis: 100–300 MB.
- Reverse proxy and system services: 100–250 MB.

These estimates must be validated by load testing.

### 18.2 Performance Targets

- Cached API p95 below 300 ms.
- Standard API p95 below 800 ms.
- Internal complex order operations below 2 seconds when not blocked by external providers.
- SSE propagation below 3 seconds.
- Dashboard initial load below 3 seconds on a typical connection.
- Background job starts within 10 seconds.

### 18.3 Planning Capacity

- Up to 20 active tenants.
- Up to 100 tenant administrative users.
- Up to 1,000 orders per day platform-wide.
- Up to 100 concurrent storefront users.
- Up to 50 concurrent SSE connections.
- Up to 10,000 active products.

These are planning assumptions, not guaranteed limits.

### 18.4 Reliability and Security

Target availability is 99.5% monthly for MVP. Services must restart automatically, jobs must be retryable, backups must run daily, and transactional operations must remain idempotent.

Minimum security includes HTTPS, secure password hashing, validation, encoding, CSRF protection where applicable, rate limiting, tenant-aware authorization, secret management, webhook validation, parameterized database access, security headers, audit logging, and dependency scanning.

### 18.5 Observability

Structured logs, correlation IDs, tenant context, latency, error rates, failed jobs, integration failures, SSE connections, database and Redis health, CPU, memory, and disk usage must be observable.

---

## 19. Consistency and Idempotency

Strong transactional guarantees are required for order creation, inventory reservation and release, payment confirmation, fulfillment transitions, shipment creation, and refund initiation.

Idempotency is required for checkout, payment callbacks, Biteship webhooks, shipment creation, inventory deduction, refund requests, and retryable jobs.

Dashboard aggregates, reports, notifications, search indexes, tracking views, and SSE refresh events may be eventually consistent.

---

## 20. Core Business Flows

### 20.1 Tenant Onboarding

Superadmin creates tenant; system provisions one store and warehouse; tenant owner is created or invited; store, warehouse, shipping, and products are configured; store is activated.

### 20.2 Checkout

Customer creates cart, enters address, receives Biteship rates, selects delivery, submits order idempotently, inventory is reserved, payment begins, and dashboards receive events.

### 20.3 Fulfillment

Payment is confirmed; order enters fulfillment; warehouse picks and packs; final package data is entered; Biteship shipment is created; tracking is stored; users receive status updates.

### 20.4 Delivery Tracking

Biteship webhook is validated and persisted idempotently; shipment and order are updated; event is published to Redis Streams; SSE and notifications deliver the change.

### 20.5 Cancellation

Eligibility is validated; fulfillment and shipment state are checked; reservation is released where applicable; refund begins where required; action is audited and published.

---

## 21. Deployment Model

Initial deployment consists of a reverse proxy, one Go gateway process, one Medusa process, one worker process, local PostgreSQL, local Redis, static frontend assets, and preferably external S3-compatible object storage.

Suggested containers:

1. `gateway`
2. `commerce`
3. `worker`
4. `postgres`
5. `redis`
6. `reverse-proxy`

Frontends should preferably be static builds rather than separate always-on Node.js production servers.

---

## 22. Repository Governance

`MartinZakhaev/terra-commerce-master-spec` is the authoritative source for product, functional, architecture, technical design, data, API, events, state machines, authorization, infrastructure, quality, operations, ADRs, and release manifests.

Application source code lives as Git submodules under root-level `apps/`:

```text
apps/
├── gateway/
├── commerce/
├── superadmin-web/
├── tenant-admin-web/
├── storefront-web/
└── infrastructure/
```

The master repository owns scope, business rules, architecture decisions, cross-application contracts, state machines, data definitions, security rules, deployment requirements, and release composition.

Application repositories own source code, tests, dependencies, builds, and local development setup.

Each master-spec release must pin all submodules to explicit commits. Cross-application changes must update specifications and contracts before or together with implementation.

---

## 23. MVP Acceptance Criteria

The MVP is accepted when:

1. Superadmin can create, activate, suspend, and inspect tenants.
2. Tenant owner can manage users and permissions.
3. Tenant admin can configure store and warehouse.
4. Products can be created and published.
5. Customers can browse, cart, request rates, and order.
6. Inventory reservation is safe and tenant isolated.
7. Tenant staff can process orders.
8. Warehouse staff can pick and pack.
9. Biteship shipments can be created and tracked.
10. Webhooks update delivery state idempotently.
11. Dashboards receive authenticated tenant-scoped SSE events.
12. Suspended tenants cannot perform normal operations.
13. Critical actions are audited.
14. Cross-tenant access tests pass.
15. The agreed load test passes on 2 vCPU and 4 GB RAM.
16. Backup and restore procedures are tested.
17. Authoritative documents are stored in the master-spec repository.
18. Application repositories are linked through `apps/` submodules.

---

## 24. Locked MVP Decisions

1. Multitenant SaaS e-commerce.
2. Go gateway.
3. Medusa.js v2 commerce engine.
4. PostgreSQL shared database and schema.
5. Redis Streams for internal events.
6. SSE for real-time UI updates.
7. Biteship for delivery.
8. One tenant, one store, one warehouse.
9. Modular monolith.
10. Initial server target: 2 vCPU and 4 GB RAM.
11. Master source of truth: `terra-commerce-master-spec`.
12. Application repositories under root-level `apps/` as submodules.
13. OMS and WMS UI modules inside tenant admin.
14. Static frontend builds preferred.
15. External S3-compatible object storage preferred.

---

## 25. Remaining Decisions

To be resolved in dependent specifications:

- Frontend framework.
- Authentication implementation.
- Payment provider.
- Biteship credential ownership.
- Tenant URL model.
- Object storage and email provider.
- Tax and guest-checkout policy.
- Reservation expiration.
- Order-number format.
- Data, log, audit, and backup retention.
- Return and refund policy.
- Subscription billing implementation.

---

## 26. Required Follow-Up Documents

1. Functional Specification Document.
2. System Architecture Document.
3. Technical Design Document.
4. Database Design and Data Dictionary.
5. OpenAPI Contract.
6. Event, SSE, and Webhook Contracts.
7. State Machine Specification.
8. Authorization Matrix.
9. Biteship Integration Specification.
10. Infrastructure and Deployment Specification.
11. Test Strategy.
12. Security and Threat Model.
13. Operational Runbook.
14. Architecture Decision Records.
15. Release Manifest Specification.

---

## 27. Definition of Done

A feature is complete when requirements, tenant isolation, authorization, errors, audit, events, SSE, idempotency, API documentation, tests, monitoring, performance review, acceptance criteria, master-spec updates, and release submodule references are complete where applicable.

---

## 28. Approval

**Status:** Final  
**Version:** 1.0.0  
**Approved Scope:** MVP  
**Change Control:** Material scope changes require a new PRD version and synchronized updates to dependent documents.
