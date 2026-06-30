# Database Design — Terra Commerce

**Document ID:** DATA-TC-001  
**Version:** 1.0.0  
**Status:** Draft  
**Owner:** Technical Lead and Backend Lead  
**Last Updated:** 2026-06-30  
**Upstream:** PRD v1.0.0 Final, FSD v1.0.1 Draft, System Architecture v1.0.0 Final, State Machine v1.0.0 Final, ADR-0001 through ADR-0007

## 1. Purpose

This document defines the logical and physical PostgreSQL design for Terra Commerce MVP, including entity ownership, tenant isolation, keys, constraints, indexes, state persistence, auditability, event durability, idempotency, retention boundaries, and migration rules.

## 2. Design Principles

1. Shared database and shared schema.
2. Every tenant-owned record contains `tenant_id` or a documented enforceable ownership path.
3. UUID primary keys are preferred for Terra Commerce-owned tables.
4. Timestamps use `timestamptz` in UTC.
5. Money uses integer minor units plus ISO currency code.
6. Mutable state aggregates use `version` for optimistic concurrency.
7. Historical business snapshots are immutable.
8. Soft deletion is used only where restoration or historical references require it.
9. Critical state changes create append-only transition history and durable outbox records.
10. Medusa-owned tables are extended through supported modules and references rather than undocumented direct modification.

## 3. Schema Ownership

Recommended logical namespaces:

- `platform` — tenants, plans, global settings, feature flags, memberships, roles, permissions.
- `commerce_ext` — Terra Commerce extensions around Medusa orders, customers, inventory, OMS, and WMS.
- `integration` — Biteship, email, payment, webhook, and provider idempotency.
- `realtime` — notifications, SSE replay metadata where persisted.
- `audit` — append-only audit and transition history.
- `ops` — jobs, dead letters, reports, outbox, migration metadata.

A single PostgreSQL schema may be used physically for MVP if framework constraints require it, but table naming and ownership must preserve these logical boundaries.

## 4. Tenant Ownership Rules

### Direct ownership

Tables containing `tenant_id` directly:

- stores
- warehouses
- tenant_memberships
- tenant_role_assignments
- tenant_feature_overrides
- tenant_usage_counters
- product_tenant_links
- customer_tenant_profiles
- order_extensions
- inventory_items
- inventory_reservations
- inventory_movements
- receiving_records
- picking_tasks
- packing_tasks
- warehouse_exceptions
- shipments
- provider_webhook_events
- notifications
- generated_reports
- audit_logs
- state_transitions
- outbox_events

### Indirect ownership

A child table may omit `tenant_id` only when:

- Its parent has `tenant_id`.
- A non-null foreign key enforces parent ownership.
- Queries cannot legally access it without joining through the tenant-owned parent.
- The ownership path is documented in the Data Dictionary.

### Global tables

- plans
- permissions
- feature_definitions
- global_settings
- system_health_snapshots

Global tables require global authorization and must not be exposed through tenant APIs without filtering.

## 5. Core Entity Model

```text
plans 1---* tenants 1---1 stores
                  |
                  1---1 warehouses
                  |
                  *---* users via tenant_memberships
                  |
                  1---* customer_tenant_profiles
                  |
                  1---* order_extensions
                  |
                  1---* inventory_items
                  |
                  1---* shipments
                  |
                  1---* notifications

order_extensions 1---* inventory_reservations
order_extensions 1---* picking_tasks
order_extensions 1---* packing_tasks
order_extensions 1---* shipments
order_extensions 1---* return_requests
order_extensions 1---* refund_requests

inventory_items 1---* inventory_movements
warehouses 1---* receiving_records
```

## 6. Global Platform Tables

### 6.1 `plans`

Purpose: subscription-plan definitions and limits.

Key columns:

- `id uuid primary key`
- `code varchar(64) unique not null`
- `name varchar(128) not null`
- `status varchar(32) not null`
- `trial_days integer not null default 0`
- `limits jsonb not null`
- `enforcement_policy jsonb not null`
- `created_at timestamptz not null`
- `updated_at timestamptz not null`
- `archived_at timestamptz null`

Constraints:

- `trial_days >= 0`
- status in `active`, `inactive`, `archived`

Indexes:

- unique `code`
- `status`

### 6.2 `feature_definitions`

- `id uuid primary key`
- `key varchar(128) unique not null`
- `description text`
- `default_enabled boolean not null`
- `created_at`, `updated_at`

### 6.3 `global_settings`

- `key varchar(128) primary key`
- `value jsonb not null`
- `is_secret boolean not null default false`
- `version integer not null default 1`
- `updated_by uuid null`
- `updated_at timestamptz not null`

Secret values must be encrypted or stored outside the table with only a reference persisted.

## 7. Tenant and Identity Tables

### 7.1 `tenants`

- `id uuid primary key`
- `plan_id uuid not null references plans(id)`
- `slug varchar(100) unique not null`
- `display_name varchar(180) not null`
- `legal_name varchar(220) null`
- `status varchar(32) not null`
- `timezone varchar(64) not null`
- `currency char(3) not null`
- `version integer not null default 1`
- `created_at`, `updated_at`
- `suspended_at`, `deactivated_at`, `archived_at` nullable

Status constraint follows the final Tenant state machine.

### 7.2 `stores`

- `id uuid primary key`
- `tenant_id uuid not null unique references tenants(id)`
- `name varchar(180) not null`
- `description text`
- `contact_email varchar(320)`
- `contact_phone varchar(32)`
- `address jsonb`
- `branding jsonb not null default '{}'`
- `settings jsonb not null default '{}'`
- `created_at`, `updated_at`

One-to-one tenant relationship is enforced with unique `tenant_id`.

### 7.3 `warehouses`

- `id uuid primary key`
- `tenant_id uuid not null unique references tenants(id)`
- `name varchar(180) not null`
- `status varchar(32) not null`
- `address jsonb not null`
- `contact_name varchar(180)`
- `contact_phone varchar(32)`
- `biteship_origin_area_id varchar(128)`
- `operating_hours jsonb`
- `version integer not null default 1`
- `created_at`, `updated_at`

### 7.4 `users`

Global identity record:

- `id uuid primary key`
- `email varchar(320) unique not null`
- `password_hash text null`
- `status varchar(32) not null`
- `last_login_at timestamptz null`
- `created_at`, `updated_at`

### 7.5 `tenant_memberships`

- `id uuid primary key`
- `tenant_id uuid not null references tenants(id)`
- `user_id uuid not null references users(id)`
- `status varchar(32) not null`
- `is_owner boolean not null default false`
- `invited_by uuid null references users(id)`
- `invited_at`, `accepted_at`, `deactivated_at` nullable
- `created_at`, `updated_at`

Constraints:

- unique `(tenant_id, user_id)`
- application rule: at least one active owner remains

### 7.6 `roles`, `permissions`, `role_permissions`, `tenant_role_assignments`

Seeded role and permission model.

Important constraints:

- permission key unique, for example `order.cancel`
- unique `(role_id, permission_id)`
- unique `(tenant_id, membership_id, role_id)`
- global roles cannot be assigned through tenant tables

### 7.7 `tenant_feature_overrides`

- `tenant_id`
- `feature_id`
- `enabled boolean`
- `effective_from`, `effective_until`
- composite primary key `(tenant_id, feature_id)`

### 7.8 `tenant_usage_counters`

- `tenant_id`
- `metric_key`
- `period_start`
- `period_end`
- `current_value bigint`
- `version integer`
- composite unique `(tenant_id, metric_key, period_start)`

## 8. Commerce Integration Tables

Medusa remains owner of core product, variant, cart, customer, order, payment, and fulfillment tables. Terra Commerce uses extension/link tables rather than relying on undocumented internal columns.

### 8.1 `product_tenant_links`

- `tenant_id uuid not null`
- `medusa_product_id text not null`
- `status varchar(32) not null`
- `created_at`, `updated_at`
- primary key `(tenant_id, medusa_product_id)`

### 8.2 `variant_tenant_links`

- `tenant_id uuid not null`
- `medusa_variant_id text not null`
- `sku varchar(128) not null`
- `weight_grams integer null`
- `length_mm integer null`
- `width_mm integer null`
- `height_mm integer null`
- `created_at`, `updated_at`

Constraints:

- unique `(tenant_id, sku)`
- dimensions and weight positive when present

### 8.3 `customer_tenant_profiles`

- `id uuid primary key`
- `tenant_id uuid not null`
- `medusa_customer_id text not null`
- `email_normalized varchar(320) not null`
- `status varchar(32) not null`
- `created_at`, `updated_at`

Unique policy:

- `(tenant_id, medusa_customer_id)` unique
- `(tenant_id, email_normalized)` unique for MVP tenant-scoped identity

### 8.4 `order_extensions`

- `id uuid primary key`
- `tenant_id uuid not null`
- `medusa_order_id text not null`
- `order_number varchar(64) not null`
- `business_status varchar(48) not null`
- `payment_status varchar(48) not null`
- `fulfillment_status varchar(48) not null`
- `delivery_status varchar(48) null`
- `customer_id uuid null references customer_tenant_profiles(id)`
- `currency char(3) not null`
- `subtotal_minor bigint not null`
- `discount_minor bigint not null default 0`
- `tax_minor bigint not null default 0`
- `shipping_minor bigint not null default 0`
- `total_minor bigint not null`
- `billing_address_snapshot jsonb`
- `shipping_address_snapshot jsonb not null`
- `pricing_snapshot jsonb not null`
- `version integer not null default 1`
- `created_at`, `updated_at`, `completed_at`, `cancelled_at`

Constraints:

- unique `(tenant_id, medusa_order_id)`
- unique `(tenant_id, order_number)`
- monetary values non-negative
- statuses constrained to final state-machine values

### 8.5 `order_notes`

Indirectly owned through `order_id`.

- `id uuid primary key`
- `order_id uuid not null references order_extensions(id)`
- `author_user_id uuid not null`
- `body text not null`
- `visibility varchar(24) not null default 'internal'`
- `created_at timestamptz not null`

## 9. Inventory and WMS Tables

### 9.1 `inventory_items`

- `id uuid primary key`
- `tenant_id uuid not null`
- `warehouse_id uuid not null`
- `medusa_variant_id text not null`
- `sku varchar(128) not null`
- `physical_qty integer not null default 0`
- `reserved_qty integer not null default 0`
- `incoming_qty integer not null default 0`
- `damaged_qty integer not null default 0`
- `low_stock_threshold integer not null default 0`
- `version integer not null default 1`
- `created_at`, `updated_at`

Constraints:

- unique `(tenant_id, warehouse_id, medusa_variant_id)`
- all quantities non-negative
- `reserved_qty <= physical_qty`
- available quantity is computed as `physical_qty - reserved_qty`

### 9.2 `inventory_reservations`

- `id uuid primary key`
- `tenant_id uuid not null`
- `inventory_item_id uuid not null references inventory_items(id)`
- `order_id uuid not null references order_extensions(id)`
- `quantity integer not null`
- `status varchar(32) not null`
- `expires_at timestamptz null`
- `idempotency_key varchar(128) not null`
- `version integer not null default 1`
- `created_at`, `updated_at`, `released_at`, `consumed_at`

Constraints:

- quantity > 0
- unique `(tenant_id, idempotency_key)`
- status follows reservation state machine

### 9.3 `inventory_movements`

Append-only.

- `id uuid primary key`
- `tenant_id uuid not null`
- `warehouse_id uuid not null`
- `inventory_item_id uuid not null`
- `movement_type varchar(48) not null`
- `before_qty integer not null`
- `delta_qty integer not null`
- `after_qty integer not null`
- `reference_type varchar(48)`
- `reference_id varchar(128)`
- `reason text`
- `actor_user_id uuid null`
- `correlation_id uuid null`
- `created_at timestamptz not null`

Movement types include initial, receiving, manual_add, manual_deduct, reservation, release, fulfillment, return, damage, correction.

### 9.4 `receiving_records`

- `id uuid primary key`
- `tenant_id uuid not null`
- `warehouse_id uuid not null`
- `status varchar(32) not null`
- `reference_no varchar(64)`
- `received_by uuid null`
- `version integer not null default 1`
- `created_at`, `completed_at`

### 9.5 `receiving_lines`

- `id uuid primary key`
- `receiving_id uuid not null references receiving_records(id)`
- `inventory_item_id uuid not null`
- `expected_qty integer null`
- `received_qty integer not null default 0`
- `damaged_qty integer not null default 0`
- `rejected_qty integer not null default 0`

### 9.6 `picking_tasks`

- `id uuid primary key`
- `tenant_id uuid not null`
- `order_id uuid not null unique`
- `warehouse_id uuid not null`
- `status varchar(32) not null`
- `assigned_user_id uuid null`
- `version integer not null default 1`
- `created_at`, `started_at`, `completed_at`, `cancelled_at`

### 9.7 `picking_lines`

- `id uuid primary key`
- `picking_task_id uuid not null`
- `inventory_item_id uuid not null`
- `required_qty integer not null`
- `picked_qty integer not null default 0`
- `status varchar(32) not null`

### 9.8 `packing_tasks`

- `id uuid primary key`
- `tenant_id uuid not null`
- `order_id uuid not null unique`
- `status varchar(32) not null`
- `assigned_user_id uuid null`
- `package_weight_grams integer null`
- `package_length_mm integer null`
- `package_width_mm integer null`
- `package_height_mm integer null`
- `version integer not null default 1`
- `created_at`, `started_at`, `completed_at`, `cancelled_at`

### 9.9 `warehouse_exceptions`

- `id uuid primary key`
- `tenant_id uuid not null`
- `order_id uuid not null`
- `task_type varchar(24) not null`
- `task_id uuid not null`
- `exception_type varchar(48) not null`
- `status varchar(32) not null`
- `details jsonb not null`
- `resolution varchar(48) null`
- `resolved_by uuid null`
- `created_at`, `resolved_at`

## 10. Shipment, Return, and Refund Tables

### 10.1 `shipments`

- `id uuid primary key`
- `tenant_id uuid not null`
- `order_id uuid not null unique`
- `provider varchar(32) not null default 'biteship'`
- `provider_order_id varchar(128) null`
- `courier_code varchar(64)`
- `service_code varchar(64)`
- `tracking_number varchar(128)`
- `label_url text`
- `shipping_cost_minor bigint not null default 0`
- `status varchar(48) not null`
- `idempotency_key varchar(128) not null`
- `version integer not null default 1`
- `created_at`, `updated_at`, `delivered_at`, `cancelled_at`

Constraints:

- unique `(tenant_id, idempotency_key)`
- unique `(provider, provider_order_id)` where provider_order_id is not null
- status follows shipment state machine

### 10.2 `shipment_tracking_events`

Append-only.

- `id uuid primary key`
- `shipment_id uuid not null`
- `provider_event_id varchar(128) null`
- `provider_status varchar(128)`
- `canonical_status varchar(48) not null`
- `event_time timestamptz null`
- `received_at timestamptz not null`
- `location jsonb`
- `description text`
- `raw_reference jsonb`

### 10.3 `return_requests`

- `id uuid primary key`
- `tenant_id uuid not null`
- `order_id uuid not null`
- `status varchar(32) not null`
- `reason_code varchar(64) not null`
- `customer_notes text`
- `reviewed_by uuid null`
- `version integer not null default 1`
- `created_at`, `updated_at`, `completed_at`

### 10.4 `return_items`

- `id uuid primary key`
- `return_request_id uuid not null`
- `order_item_reference varchar(128) not null`
- `quantity integer not null`
- `disposition varchar(48) null`

### 10.5 `refund_requests`

- `id uuid primary key`
- `tenant_id uuid not null`
- `order_id uuid not null`
- `return_request_id uuid null`
- `status varchar(32) not null`
- `requested_minor bigint not null`
- `completed_minor bigint not null default 0`
- `currency char(3) not null`
- `provider_reference varchar(128) null`
- `idempotency_key varchar(128) not null`
- `version integer not null default 1`
- `created_at`, `updated_at`, `completed_at`

Constraints:

- requested and completed values non-negative
- completed <= requested
- unique `(tenant_id, idempotency_key)`

## 11. Integration and Reliability Tables

### 11.1 `provider_webhook_events`

- `id uuid primary key`
- `tenant_id uuid null`
- `provider varchar(32) not null`
- `provider_event_id varchar(160) not null`
- `event_type varchar(128)`
- `payload_hash varchar(128) not null`
- `status varchar(32) not null`
- `received_at timestamptz not null`
- `processed_at timestamptz null`
- `failure_reason text null`

Constraint: unique `(provider, provider_event_id)`.

### 11.2 `idempotency_records`

- `id uuid primary key`
- `tenant_id uuid null`
- `scope varchar(64) not null`
- `idempotency_key varchar(160) not null`
- `request_hash varchar(128) not null`
- `response_status integer null`
- `response_body jsonb null`
- `resource_type varchar(64) null`
- `resource_id varchar(128) null`
- `expires_at timestamptz null`
- `created_at`, `completed_at`

Unique `(tenant_id, scope, idempotency_key)` using a null-safe global strategy.

### 11.3 `outbox_events`

Durable event publication table.

- `id uuid primary key`
- `tenant_id uuid null`
- `aggregate_type varchar(64) not null`
- `aggregate_id varchar(128) not null`
- `event_type varchar(128) not null`
- `schema_version integer not null`
- `payload jsonb not null`
- `correlation_id uuid null`
- `status varchar(24) not null default 'pending'`
- `attempt_count integer not null default 0`
- `available_at timestamptz not null`
- `published_at timestamptz null`
- `created_at timestamptz not null`

Indexes:

- `(status, available_at)`
- `(aggregate_type, aggregate_id)`

### 11.4 `background_jobs` and `dead_letter_jobs`

Persist operational job state when the selected worker implementation requires database durability beyond Redis Streams.

Fields include tenant, job type, status, idempotency key, attempts, payload reference, next attempt, error metadata, and timestamps.

## 12. Notification, Reporting, and Audit Tables

### 12.1 `notifications`

- `id uuid primary key`
- `tenant_id uuid not null`
- `recipient_user_id uuid null`
- `recipient_customer_id uuid null`
- `type varchar(64) not null`
- `title varchar(220) not null`
- `body text not null`
- `data jsonb not null default '{}'`
- `read_at timestamptz null`
- `created_at timestamptz not null`

Exactly one recipient type must be present.

### 12.2 `generated_reports`

- `id uuid primary key`
- `tenant_id uuid null`
- `requested_by uuid not null`
- `report_type varchar(64) not null`
- `parameters jsonb not null`
- `status varchar(32) not null`
- `object_key text null`
- `expires_at timestamptz null`
- `created_at`, `completed_at`

### 12.3 `audit_logs`

Append-only.

- `id uuid primary key`
- `tenant_id uuid null`
- `actor_user_id uuid null`
- `effective_tenant_id uuid null`
- `action varchar(128) not null`
- `resource_type varchar(64) not null`
- `resource_id varchar(128) null`
- `before_data jsonb null`
- `after_data jsonb null`
- `ip_address inet null`
- `user_agent text null`
- `correlation_id uuid null`
- `created_at timestamptz not null`

### 12.4 `state_transitions`

Append-only.

- `id uuid primary key`
- `tenant_id uuid null`
- `aggregate_type varchar(64) not null`
- `aggregate_id varchar(128) not null`
- `version_before integer not null`
- `version_after integer not null`
- `state_before varchar(64) not null`
- `state_after varchar(64) not null`
- `actor_user_id uuid null`
- `reason text null`
- `idempotency_key varchar(160) null`
- `correlation_id uuid null`
- `created_at timestamptz not null`

## 13. Indexing Strategy

Mandatory patterns:

- Every tenant-owned high-volume table starts composite indexes with `tenant_id`.
- Foreign keys used in joins are indexed.
- Status and queue tables index `(status, available_at)` or `(tenant_id, status, created_at)`.
- Order lookups index `(tenant_id, order_number)` and `(tenant_id, created_at desc)`.
- Inventory indexes cover `(tenant_id, warehouse_id, sku)`.
- Shipment indexes cover `(tenant_id, tracking_number)` and `(provider, provider_order_id)`.
- Audit and transition history index `(tenant_id, resource_type, resource_id, created_at desc)`.

Avoid excessive indexes on low-write-value columns due to the 2 vCPU / 4 GB RAM target.

## 14. Concurrency and Locking

- Stateful aggregates use `version` and atomic `update ... where id = ? and version = ?`.
- Inventory reservation and deduction use row-level locking or equivalent atomic updates.
- Usage counters use atomic increments.
- Outbox insertion occurs in the same transaction as the business state change.
- Long user transactions are prohibited.
- Provider callbacks must lock or compare versions before transitions.

## 15. Soft Delete and Retention

Soft delete or archival is allowed for tenants, plans, products through Medusa-supported mechanisms, users/memberships, and generated reports.

Never hard-delete records referenced by:

- Orders.
- Inventory movements.
- Shipments.
- Audit logs.
- State transitions.
- Refunds.

Retention durations remain an operations-policy decision, but the schema must support partitioning or archival for audit logs, transitions, webhook events, tracking events, outbox, and job history.

## 16. Migration Strategy

- Use version-controlled forward migrations.
- Every production migration is reviewed for tenant impact, lock duration, data backfill, rollback, and Medusa compatibility.
- Large backfills run in batches.
- Add nullable columns before enforcing non-null where existing data requires backfill.
- Breaking enum or status changes require state-machine and contract version updates.
- Migrations must not depend on application startup ordering without explicit orchestration.

## 17. Backup and Recovery

- PostgreSQL is authoritative and backed up outside the application VPS.
- Restore tests validate schema, constraints, and representative tenant data.
- Redis is not the sole recovery source for business truth.
- Object-storage keys are recoverable according to provider retention.
- Migration rollback and forward-fix instructions accompany releases.

## 18. Open Implementation Decisions

- Exact Medusa v2 table names and extension points.
- PostgreSQL enum types versus check constraints.
- Native Row-Level Security adoption.
- Table partitioning thresholds.
- Authentication/session table implementation.
- Payment-provider-specific tables.
- Reservation expiration duration.
- Order-number generation algorithm.
- Data retention periods.

## 19. Acceptance Criteria

1. Every tenant-owned domain has enforceable tenant ownership.
2. All final state-machine states can be persisted and versioned.
3. Critical state changes support transition history and durable outbox publication.
4. Inventory quantities cannot become invalid through normal writes.
5. Duplicate webhook, checkout, shipment, and refund operations are prevented.
6. Historical orders remain immutable through snapshots.
7. Foreign keys and indexes support primary access paths.
8. PostgreSQL remains the authoritative recovery source.
9. Medusa ownership is respected without undocumented schema coupling.
10. The design remains viable for the MVP resource target.

## Change Summary

Initial detailed database design derived from the approved functional, architecture, and state-machine specifications.
