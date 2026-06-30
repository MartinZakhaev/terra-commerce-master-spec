# Database Design — Terra Commerce

**Document ID:** DATA-TC-001  
**Version:** 1.0.0  
**Status:** Final  
**Owner:** Technical Lead and Backend Lead  
**Last Updated:** 2026-06-30  
**Upstream:** PRD v1.0.0 Final, FSD v1.0.1 Draft, System Architecture v1.0.0 Final, State Machine v1.0.0 Final, ADR-0001 through ADR-0007  
**Audit:** `reviews/Database_Design_Data_Dictionary_Audit_v1.0.0.md`

## 1. Purpose

This document defines the approved PostgreSQL design for Terra Commerce MVP, including entity ownership, tenant isolation, keys, constraints, indexes, state persistence, auditability, idempotency, durable event publication, retention, migration, and recovery.

## 2. Design Principles

1. Shared database and shared schema.
2. Every tenant-owned record contains `tenant_id` or an enforceable ownership path.
3. UUID primary keys are preferred for Terra Commerce-owned tables.
4. Timestamps use `timestamptz` in UTC.
5. Money uses integer minor units and ISO currency codes.
6. Stateful aggregates use `version` for atomic optimistic concurrency.
7. Historical order snapshots are immutable.
8. Critical state changes create append-only transition history and outbox records in the same transaction.
9. Medusa-owned tables are referenced or extended only through supported extension points.
10. PostgreSQL is the authoritative recovery source.

## 3. Logical Ownership Domains

- `platform`: tenants, plans, settings, features, identities, memberships, roles, permissions.
- `commerce_ext`: Terra Commerce references and projections around Medusa products, customers, orders, payments, fulfillment, OMS, and WMS.
- `integration`: shipments, provider webhooks, idempotency, provider references.
- `realtime`: notifications and persisted replay metadata where needed.
- `audit`: audit logs and state transitions.
- `ops`: outbox, jobs, dead letters, generated reports.

These may share one physical PostgreSQL schema during MVP, but ownership boundaries remain mandatory.

## 4. Tenant-Safe Foreign-Key Strategy

Every tenant-owned parent table must expose:

```sql
UNIQUE (tenant_id, id)
```

Tenant-owned children with their own `tenant_id` use composite foreign keys:

```sql
FOREIGN KEY (tenant_id, parent_id)
REFERENCES parent_table (tenant_id, id)
```

This pattern applies wherever practical to orders, warehouses, inventory items, reservations, receiving records, picking tasks, packing tasks, shipments, returns, refunds, notifications, audit records, transition records, and outbox aggregates.

Children without `tenant_id` are permitted only when their parent foreign key is non-null, ownership is unambiguous, and access is always through the parent repository.

## 5. Global Platform Tables

### `plans`

- `id uuid primary key`
- `code varchar(64) unique not null`
- `name varchar(128) not null`
- `status varchar(32) not null`
- `trial_days integer not null default 0`
- `limits jsonb not null`
- `enforcement_policy jsonb not null`
- `created_at`, `updated_at`, `archived_at`

Checks: non-negative trial days; allowed statuses `active`, `inactive`, `archived`.

### `feature_definitions`

- `id uuid primary key`
- `key varchar(128) unique not null`
- `description text`
- `default_enabled boolean not null`
- timestamps

### `global_settings`

- `key varchar(128) primary key`
- `value jsonb not null`
- `is_secret boolean not null default false`
- `version integer not null default 1`
- `updated_by uuid null`
- `updated_at timestamptz not null`

Secrets are encrypted or externalized; raw secret values are never returned after storage.

## 6. Tenant and Identity Tables

### `tenants`

- `id uuid primary key`
- `plan_id uuid not null references plans(id)`
- `slug varchar(100) unique not null`
- `display_name varchar(180) not null`
- `legal_name varchar(220)`
- `status varchar(32) not null`
- `timezone varchar(64) not null`
- `currency char(3) not null`
- `version integer not null default 1`
- lifecycle timestamps

Status follows the final Tenant state machine.

### `stores`

- `id uuid primary key`
- `tenant_id uuid not null unique references tenants(id)`
- identity, contact, address, branding, and settings fields
- timestamps

### `warehouses`

- `id uuid primary key`
- `tenant_id uuid not null unique references tenants(id)`
- `name`, `status`, `address`, contact fields
- `biteship_origin_area_id`
- `operating_hours jsonb`
- `version`
- timestamps

### `users`

- global identity ID
- globally unique normalized email
- password hash or external identity reference
- account status and login timestamps

### `tenant_memberships`

- `id`, `tenant_id`, `user_id`
- invitation and activation lifecycle
- `is_owner`
- unique `(tenant_id, user_id)`

Application and database workflows must preserve at least one active tenant owner.

### Authorization tables

- `roles`
- `permissions`
- `role_permissions`
- `tenant_role_assignments`

Permission keys use action notation such as `order.cancel`. Global roles cannot be assigned through tenant-scoped assignment tables.

### `tenant_feature_overrides`

Composite primary key `(tenant_id, feature_id)` with effective dates.

### `tenant_usage_counters`

Unique `(tenant_id, metric_key, period_start)` and atomic versioned increments.

## 7. Medusa Integration Boundaries

Medusa owns core product, variant, customer, cart, order, payment, and fulfillment tables. Terra Commerce does not assume undocumented Medusa internal names in contracts.

Terra Commerce-owned extension tables use stable external references:

### `product_tenant_links`

Primary key `(tenant_id, medusa_product_id)` and tenant-scoped product status.

### `variant_tenant_links`

- tenant and Medusa variant reference
- tenant-unique SKU
- shipping weight and dimensions

### `customer_tenant_profiles`

- tenant and Medusa customer reference
- tenant-scoped normalized email
- customer status

Unique `(tenant_id, medusa_customer_id)` and `(tenant_id, email_normalized)`.

## 8. Order, Payment, and Fulfillment Persistence

### `order_extensions`

Stores Terra Commerce business projection and immutable snapshots:

- `id`, `tenant_id`, `medusa_order_id`, `order_number`
- business, payment, fulfillment, and delivery projections
- customer reference
- currency and monetary snapshots
- billing and shipping address snapshots
- pricing snapshot
- `version`
- lifecycle timestamps

Unique `(tenant_id, medusa_order_id)` and `(tenant_id, order_number)`.

The projection columns are optimized read values and are not the sole concurrency authority for payment and fulfillment sub-lifecycles.

### `payment_state_extensions`

- `id uuid primary key`
- `tenant_id uuid not null`
- `order_id uuid not null`
- `medusa_payment_reference text null`
- `status varchar(48) not null`
- `authorized_minor bigint not null default 0`
- `captured_minor bigint not null default 0`
- `refunded_minor bigint not null default 0`
- `currency char(3) not null`
- `version integer not null default 1`
- provider and lifecycle timestamps

Unique `(tenant_id, order_id)`. Amount checks enforce captured <= authorized or payable policy, and refunded <= captured.

### `fulfillment_state_extensions`

- `id uuid primary key`
- `tenant_id uuid not null`
- `order_id uuid not null`
- `medusa_fulfillment_reference text null`
- `status varchar(48) not null`
- `version integer not null default 1`
- timestamps

Unique `(tenant_id, order_id)`.

### `order_notes`

Indirectly tenant-owned through `order_id`; internal visibility only for MVP.

## 9. Inventory and WMS

### Inventory quantity semantics

- `physical_qty`: saleable on-hand stock only; excludes damaged stock.
- `reserved_qty`: quantity held by Active reservations.
- `available_qty`: computed as `physical_qty - reserved_qty`.
- `incoming_qty`: expected but not yet accepted stock.
- `damaged_qty`: physically present but unavailable stock.

All quantities must remain non-negative, and `reserved_qty <= physical_qty`.

### `inventory_items`

Tenant, warehouse, Medusa variant, SKU, quantity buckets, threshold, version, timestamps.

Unique `(tenant_id, warehouse_id, medusa_variant_id)` and `(tenant_id, warehouse_id, sku)`.

### `inventory_reservations`

Tenant-safe references to inventory item and order, positive quantity, reservation status, expiration, idempotency key, version, timestamps.

### `inventory_movements`

Append-only movement ledger with:

- quantity bucket changed: `physical`, `reserved`, `incoming`, or `damaged`
- before, delta, and after values
- movement type
- reference, actor, reason, correlation ID

Receiving rules:

- accepted quantity decreases `incoming_qty` where applicable and increases `physical_qty`
- damaged quantity decreases `incoming_qty` where applicable and increases `damaged_qty`
- rejected quantity decreases `incoming_qty` where applicable without increasing physical or damaged stock

### Receiving

- `receiving_records`
- `receiving_lines`

Completion is versioned and idempotent.

### Picking

- `picking_tasks`
- `picking_lines`

One picking task per order in MVP.

### Packing

- `packing_tasks`

One packing task and one package per order in MVP. Completion requires positive weight and dimensions.

### `warehouse_exceptions`

Stores picking/packing exception type, evidence, status, resolution, actor, and timestamps.

## 10. Shipment, Return, and Refund

### `shipments`

Tenant-safe one-to-one order reference for MVP, provider identifiers, courier/service, tracking, label reference, shipping cost, canonical status, idempotency key, version, timestamps.

### `shipment_tracking_events`

Append-only provider and canonical tracking history. Duplicate provider event IDs are rejected where available.

### `return_requests` and `return_items`

Persist final Return lifecycle and item-level quantities/dispositions.

### `refund_requests`

Persist final Refund lifecycle, requested and completed amounts, currency, provider reference, idempotency key, version, timestamps.

## 11. Reliability and Integration Tables

### `provider_webhook_events`

Unique `(provider, provider_event_id)` with payload hash, processing status, resolved tenant, failure metadata, and timestamps.

### `idempotency_records`

Tenant operations use partial uniqueness:

```sql
CREATE UNIQUE INDEX uq_idempotency_tenant
ON idempotency_records (tenant_id, scope, idempotency_key)
WHERE tenant_id IS NOT NULL;

CREATE UNIQUE INDEX uq_idempotency_global
ON idempotency_records (scope, idempotency_key)
WHERE tenant_id IS NULL;
```

A reused key with a different request hash is rejected.

### `outbox_events`

Fields include:

- tenant
- aggregate type and ID
- `aggregate_version integer not null`
- event type and schema version
- payload and correlation ID
- status, attempts, availability, claim metadata, timestamps

Unique logical event constraint:

```sql
UNIQUE (aggregate_type, aggregate_id, aggregate_version, event_type)
```

Publishers claim rows using an atomic pattern such as `FOR UPDATE SKIP LOCKED`. Outbox insertion occurs in the same transaction as the state change.

### Jobs and dead letters

Database-backed job metadata may supplement Redis Streams where durable operational history is required.

## 12. Notification, Reporting, Audit, and Transitions

### `notifications`

Exactly one recipient is required:

```sql
CHECK (
  (recipient_user_id IS NOT NULL)::int +
  (recipient_customer_id IS NOT NULL)::int = 1
)
```

### `generated_reports`

Stores tenant/global scope, requester, report type, parameters, status, object key, expiry, and timestamps.

### `audit_logs`

Append-only critical-action history.

### `state_transitions`

Append-only lifecycle history with aggregate versions, states, actor, reason, idempotency key, correlation ID, and timestamp.

### Append-only enforcement

For `inventory_movements`, `shipment_tracking_events`, `audit_logs`, and `state_transitions`:

- normal application roles receive insert/select only
- update/delete are denied
- privileged maintenance is separately controlled and audited
- repository methods expose no normal mutation operation

## 13. Indexing Strategy

- Tenant-owned high-volume indexes begin with `tenant_id`.
- All frequently joined foreign keys are indexed.
- Queue tables index `(status, available_at)`.
- Orders index `(tenant_id, order_number)` and `(tenant_id, created_at desc)`.
- Inventory indexes `(tenant_id, warehouse_id, sku)`.
- Shipments index `(tenant_id, tracking_number)` and `(provider, provider_order_id)`.
- Audit and transition history index tenant, resource/aggregate, and descending time.

Index count must remain measured against write overhead and the MVP memory budget.

## 14. Concurrency and Locking

- Stateful aggregate update uses version comparison or row lock.
- Inventory reservation and deduction use row-level locking or atomic conditional updates.
- Usage counters use atomic increments.
- Outbox insertion shares the business transaction.
- Provider callbacks compare current state/version before transition.
- Long user-held transactions are prohibited.

## 15. Deletion and Retention

Never hard-delete data referenced by orders, payments, inventory movements, shipments, returns, refunds, audit logs, or state transitions.

Archival/retention policies must support future partitioning for audit logs, transitions, webhook events, tracking events, outbox events, and job history.

## 16. Migration Strategy

- Version-controlled forward migrations.
- Review lock duration, tenant impact, backfill, rollback, and Medusa compatibility.
- Large backfills are batched.
- Add nullable fields before backfill and later non-null enforcement.
- Breaking status changes require synchronized state-machine and contract versions.
- Releases include forward-fix and rollback guidance.

## 17. Backup and Recovery

- PostgreSQL backups are stored outside the application VPS.
- Restore tests validate representative tenant, order, inventory, shipment, audit, and transition data.
- Redis is not the only recovery source.
- Object-storage references follow provider retention and recovery policy.

## 18. Open Implementation Decisions

- Exact supported Medusa extension APIs and table references.
- PostgreSQL enums versus check constraints.
- Row-Level Security adoption.
- Partition thresholds.
- Authentication/session storage.
- Payment-provider-specific details.
- Reservation expiration duration.
- Order-number algorithm.
- Retention durations.

## 19. Acceptance Criteria

1. Tenant-safe foreign keys prevent cross-tenant parent references where practical.
2. Payment and fulfillment lifecycles have versioned persistence authorities.
3. Inventory bucket semantics and receiving equations are unambiguous.
4. Global and tenant idempotency keys are uniquely enforced.
5. Critical events are uniquely and durably published through the outbox.
6. Append-only tables and notification-recipient constraints are enforced.
7. All final state-machine values can be persisted and transitioned atomically.
8. Historical order data remains immutable.
9. Medusa ownership is respected.
10. PostgreSQL remains the authoritative recovery source.

## Change Summary

Final version closes all findings in `Database_Design_Data_Dictionary_Audit_v1.0.0.md`, including tenant-safe composite foreign keys, payment and fulfillment state persistence, exact inventory semantics, null-safe idempotency uniqueness, outbox ordering, and database-level immutability constraints.
