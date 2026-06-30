# Data Dictionary — Terra Commerce

**Document ID:** DATA-TC-002  
**Version:** 1.0.0  
**Status:** Draft  
**Owner:** Technical Lead and Backend Lead  
**Last Updated:** 2026-06-30  
**Upstream:** Database Design v1.0.0 Draft, State Machine v1.0.0 Final

## 1. Purpose

This dictionary defines the meaning, ownership, type, nullability, constraints, and lifecycle of Terra Commerce database fields. It covers Terra Commerce-owned tables and extension references to Medusa-owned records.

## 2. Common Conventions

| Name | Type | Meaning |
|---|---|---|
| `id` | `uuid` | Primary identifier |
| `tenant_id` | `uuid` | Owning tenant |
| `version` | `integer` | Optimistic concurrency version, starts at 1 |
| `created_at` | `timestamptz` | Creation timestamp in UTC |
| `updated_at` | `timestamptz` | Last modification timestamp in UTC |
| `archived_at` | `timestamptz` | Logical archival timestamp |
| `correlation_id` | `uuid` | Cross-service/request trace identifier |
| `idempotency_key` | `varchar` | Caller or system key that makes repeated operations return the same business result |
| `*_minor` | `bigint` | Monetary amount in currency minor units |
| `currency` | `char(3)` | ISO 4217 currency code |

## 3. Status Domains

### Tenant status

`draft`, `trial`, `active`, `past_due`, `suspended`, `deactivated`, `archived`

### Order business status

`draft`, `pending_payment`, `payment_pending_verification`, `paid`, `processing`, `ready_for_fulfillment`, `picking`, `packing`, `ready_for_shipment`, `shipped`, `delivered`, `completed`, `cancellation_requested`, `cancelled`, `return_requested`, `returned`, `refund_pending`, `refunded`, `failed`

### Payment status

`not_started`, `pending`, `pending_verification`, `authorized`, `captured`, `failed`, `cancelled`, `refund_pending`, `partially_refunded`, `refunded`

### Reservation status

`pending`, `active`, `released`, `consumed`, `expired`, `failed`

### Fulfillment status

`not_ready`, `ready`, `in_progress`, `exception`, `completed`, `cancelled`

### Picking status

`not_created`, `ready`, `in_progress`, `paused`, `exception`, `completed`, `cancelled`

### Packing status

`not_created`, `ready`, `in_progress`, `exception`, `completed`, `cancelled`

### Shipment status

`shipment_pending`, `courier_assigned`, `awaiting_pickup`, `picked_up`, `in_transit`, `out_for_delivery`, `delivered`, `delivery_failed`, `returned_to_sender`, `cancelled`, `unknown_pending_review`

### Return status

`requested`, `under_review`, `approved`, `rejected`, `in_transit`, `received`, `inspected`, `completed`, `cancelled`

### Refund status

`not_required`, `requested`, `approved`, `rejected`, `processing`, `partially_completed`, `completed`, `failed`

### Job status

`queued`, `running`, `retry_scheduled`, `completed`, `failed`, `dead_lettered`, `cancelled`

## 4. Table Dictionary

### 4.1 `plans`

| Column | Type | Null | Constraint / Meaning |
|---|---|---:|---|
| `id` | uuid | no | primary key |
| `code` | varchar(64) | no | globally unique immutable plan code |
| `name` | varchar(128) | no | display name |
| `status` | varchar(32) | no | active, inactive, archived |
| `trial_days` | integer | no | non-negative trial duration |
| `limits` | jsonb | no | product, order, user, storage, SSE, module, and report limits |
| `enforcement_policy` | jsonb | no | soft/hard enforcement by metric |
| `created_at` | timestamptz | no | creation time |
| `updated_at` | timestamptz | no | update time |
| `archived_at` | timestamptz | yes | archive time |

### 4.2 `tenants`

| Column | Type | Null | Constraint / Meaning |
|---|---|---:|---|
| `id` | uuid | no | primary key |
| `plan_id` | uuid | no | FK to plans |
| `slug` | varchar(100) | no | global unique tenant slug |
| `display_name` | varchar(180) | no | tenant-facing name |
| `legal_name` | varchar(220) | yes | registered business name |
| `status` | varchar(32) | no | tenant state-machine value |
| `timezone` | varchar(64) | no | IANA timezone |
| `currency` | char(3) | no | tenant default currency |
| `version` | integer | no | concurrency version |
| `created_at` | timestamptz | no | creation time |
| `updated_at` | timestamptz | no | update time |
| `suspended_at` | timestamptz | yes | latest suspension time |
| `deactivated_at` | timestamptz | yes | deactivation time |
| `archived_at` | timestamptz | yes | archive time |

### 4.3 `stores`

| Column | Type | Null | Constraint / Meaning |
|---|---|---:|---|
| `id` | uuid | no | primary key |
| `tenant_id` | uuid | no | unique FK; one store per tenant |
| `name` | varchar(180) | no | storefront name |
| `description` | text | yes | store description |
| `contact_email` | varchar(320) | yes | store contact email |
| `contact_phone` | varchar(32) | yes | store contact phone |
| `address` | jsonb | yes | structured store address |
| `branding` | jsonb | no | logo, colors, favicon, style settings |
| `settings` | jsonb | no | order, inventory, delivery, and notification settings |
| `created_at` | timestamptz | no | creation time |
| `updated_at` | timestamptz | no | update time |

### 4.4 `warehouses`

| Column | Type | Null | Constraint / Meaning |
|---|---|---:|---|
| `id` | uuid | no | primary key |
| `tenant_id` | uuid | no | unique FK; one warehouse per tenant |
| `name` | varchar(180) | no | warehouse display name |
| `status` | varchar(32) | no | active or inactive operational state |
| `address` | jsonb | no | structured origin address |
| `contact_name` | varchar(180) | yes | warehouse contact |
| `contact_phone` | varchar(32) | yes | warehouse phone |
| `biteship_origin_area_id` | varchar(128) | yes | provider origin reference |
| `operating_hours` | jsonb | yes | weekly schedule |
| `version` | integer | no | concurrency version |
| `created_at` | timestamptz | no | creation time |
| `updated_at` | timestamptz | no | update time |

### 4.5 `users`

| Column | Type | Null | Constraint / Meaning |
|---|---|---:|---|
| `id` | uuid | no | global identity ID |
| `email` | varchar(320) | no | globally unique normalized email |
| `password_hash` | text | yes | null for external identity-only accounts |
| `status` | varchar(32) | no | pending, active, disabled, locked |
| `last_login_at` | timestamptz | yes | last successful login |
| `created_at` | timestamptz | no | creation time |
| `updated_at` | timestamptz | no | update time |

### 4.6 `tenant_memberships`

| Column | Type | Null | Constraint / Meaning |
|---|---|---:|---|
| `id` | uuid | no | membership ID |
| `tenant_id` | uuid | no | owning tenant |
| `user_id` | uuid | no | global user |
| `status` | varchar(32) | no | invited, active, deactivated, revoked |
| `is_owner` | boolean | no | identifies tenant owner membership |
| `invited_by` | uuid | yes | inviting user |
| `invited_at` | timestamptz | yes | invitation time |
| `accepted_at` | timestamptz | yes | acceptance time |
| `deactivated_at` | timestamptz | yes | deactivation time |
| `created_at` | timestamptz | no | creation time |
| `updated_at` | timestamptz | no | update time |

Unique `(tenant_id, user_id)`.

### 4.7 `roles`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `id` | uuid | no | role ID |
| `key` | varchar(100) | no | unique role key |
| `name` | varchar(128) | no | display name |
| `scope` | varchar(24) | no | global or tenant |
| `is_system` | boolean | no | protected seeded role |

### 4.8 `permissions`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `id` | uuid | no | permission ID |
| `key` | varchar(128) | no | unique action key such as `order.cancel` |
| `description` | text | yes | human-readable meaning |

### 4.9 `tenant_usage_counters`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `tenant_id` | uuid | no | tenant |
| `metric_key` | varchar(64) | no | products, orders, users, storage, SSE, etc. |
| `period_start` | timestamptz | no | usage window start |
| `period_end` | timestamptz | no | usage window end |
| `current_value` | bigint | no | current usage |
| `version` | integer | no | atomic increment version |

### 4.10 `product_tenant_links`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `tenant_id` | uuid | no | owning tenant |
| `medusa_product_id` | text | no | Medusa product reference |
| `status` | varchar(32) | no | draft, active, inactive, archived |
| `created_at` | timestamptz | no | creation time |
| `updated_at` | timestamptz | no | update time |

### 4.11 `variant_tenant_links`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `tenant_id` | uuid | no | owning tenant |
| `medusa_variant_id` | text | no | Medusa variant reference |
| `sku` | varchar(128) | no | unique within tenant |
| `weight_grams` | integer | yes | positive shipping weight |
| `length_mm` | integer | yes | positive length |
| `width_mm` | integer | yes | positive width |
| `height_mm` | integer | yes | positive height |
| `created_at` | timestamptz | no | creation time |
| `updated_at` | timestamptz | no | update time |

### 4.12 `customer_tenant_profiles`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `id` | uuid | no | profile ID |
| `tenant_id` | uuid | no | owning tenant |
| `medusa_customer_id` | text | no | Medusa customer reference |
| `email_normalized` | varchar(320) | no | tenant-scoped unique email |
| `status` | varchar(32) | no | active, disabled, archived |
| `created_at` | timestamptz | no | creation time |
| `updated_at` | timestamptz | no | update time |

### 4.13 `order_extensions`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `id` | uuid | no | Terra order extension ID |
| `tenant_id` | uuid | no | owning tenant |
| `medusa_order_id` | text | no | Medusa order reference |
| `order_number` | varchar(64) | no | tenant-scoped human order ID |
| `business_status` | varchar(48) | no | order state-machine value |
| `payment_status` | varchar(48) | no | payment state-machine value |
| `fulfillment_status` | varchar(48) | no | fulfillment state-machine value |
| `delivery_status` | varchar(48) | yes | shipment state-machine value |
| `customer_id` | uuid | yes | customer tenant profile |
| `currency` | char(3) | no | immutable order currency |
| `subtotal_minor` | bigint | no | item subtotal |
| `discount_minor` | bigint | no | total discounts |
| `tax_minor` | bigint | no | tax snapshot |
| `shipping_minor` | bigint | no | shipping snapshot |
| `total_minor` | bigint | no | final total |
| `billing_address_snapshot` | jsonb | yes | immutable billing address |
| `shipping_address_snapshot` | jsonb | no | immutable shipping address |
| `pricing_snapshot` | jsonb | no | immutable item and adjustment snapshot |
| `version` | integer | no | concurrency version |
| `created_at` | timestamptz | no | creation time |
| `updated_at` | timestamptz | no | update time |
| `completed_at` | timestamptz | yes | completion time |
| `cancelled_at` | timestamptz | yes | cancellation time |

### 4.14 `inventory_items`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `id` | uuid | no | inventory item ID |
| `tenant_id` | uuid | no | owning tenant |
| `warehouse_id` | uuid | no | owning warehouse |
| `medusa_variant_id` | text | no | Medusa variant reference |
| `sku` | varchar(128) | no | denormalized operational SKU |
| `physical_qty` | integer | no | physically present saleable quantity before damage exclusion policy |
| `reserved_qty` | integer | no | quantity reserved by active reservations |
| `incoming_qty` | integer | no | expected inbound quantity |
| `damaged_qty` | integer | no | damaged quantity tracked separately |
| `low_stock_threshold` | integer | no | alert threshold |
| `version` | integer | no | concurrency version |
| `created_at` | timestamptz | no | creation time |
| `updated_at` | timestamptz | no | update time |

Computed available quantity: `physical_qty - reserved_qty`.

### 4.15 `inventory_reservations`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `id` | uuid | no | reservation ID |
| `tenant_id` | uuid | no | owning tenant |
| `inventory_item_id` | uuid | no | reserved inventory item |
| `order_id` | uuid | no | owning order |
| `quantity` | integer | no | reserved quantity, positive |
| `status` | varchar(32) | no | reservation state |
| `expires_at` | timestamptz | yes | expiration time once policy exists |
| `idempotency_key` | varchar(128) | no | unique tenant reservation key |
| `version` | integer | no | concurrency version |
| `created_at` | timestamptz | no | creation time |
| `updated_at` | timestamptz | no | update time |
| `released_at` | timestamptz | yes | release time |
| `consumed_at` | timestamptz | yes | consume time |

### 4.16 `inventory_movements`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `id` | uuid | no | movement ID |
| `tenant_id` | uuid | no | owning tenant |
| `warehouse_id` | uuid | no | warehouse |
| `inventory_item_id` | uuid | no | affected inventory item |
| `movement_type` | varchar(48) | no | initial, receiving, manual_add, manual_deduct, reservation, release, fulfillment, return, damage, correction |
| `before_qty` | integer | no | relevant quantity before change |
| `delta_qty` | integer | no | signed quantity change |
| `after_qty` | integer | no | relevant quantity after change |
| `reference_type` | varchar(48) | yes | order, reservation, receiving, return, manual |
| `reference_id` | varchar(128) | yes | related resource ID |
| `reason` | text | yes | mandatory for manual/correction movements |
| `actor_user_id` | uuid | yes | initiating user |
| `correlation_id` | uuid | yes | trace ID |
| `created_at` | timestamptz | no | immutable event time |

### 4.17 `receiving_records`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `id` | uuid | no | receiving record ID |
| `tenant_id` | uuid | no | owning tenant |
| `warehouse_id` | uuid | no | warehouse |
| `status` | varchar(32) | no | draft, in_progress, completed, cancelled |
| `reference_no` | varchar(64) | yes | inbound reference |
| `received_by` | uuid | yes | completing warehouse user |
| `version` | integer | no | concurrency version |
| `created_at` | timestamptz | no | creation time |
| `completed_at` | timestamptz | yes | completion time |

### 4.18 `receiving_lines`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `id` | uuid | no | line ID |
| `receiving_id` | uuid | no | parent receiving record |
| `inventory_item_id` | uuid | no | target inventory item |
| `expected_qty` | integer | yes | expected incoming quantity |
| `received_qty` | integer | no | accepted quantity |
| `damaged_qty` | integer | no | damaged quantity |
| `rejected_qty` | integer | no | rejected quantity |

### 4.19 `picking_tasks`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `id` | uuid | no | task ID |
| `tenant_id` | uuid | no | owning tenant |
| `order_id` | uuid | no | unique order reference |
| `warehouse_id` | uuid | no | warehouse |
| `status` | varchar(32) | no | picking state |
| `assigned_user_id` | uuid | yes | current picker |
| `version` | integer | no | concurrency version |
| `created_at` | timestamptz | no | creation time |
| `started_at` | timestamptz | yes | start time |
| `completed_at` | timestamptz | yes | completion time |
| `cancelled_at` | timestamptz | yes | cancellation time |

### 4.20 `packing_tasks`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `id` | uuid | no | task ID |
| `tenant_id` | uuid | no | owning tenant |
| `order_id` | uuid | no | unique order reference |
| `status` | varchar(32) | no | packing state |
| `assigned_user_id` | uuid | yes | packer |
| `package_weight_grams` | integer | yes | required before completion |
| `package_length_mm` | integer | yes | required before completion |
| `package_width_mm` | integer | yes | required before completion |
| `package_height_mm` | integer | yes | required before completion |
| `version` | integer | no | concurrency version |
| `created_at` | timestamptz | no | creation time |
| `started_at` | timestamptz | yes | start time |
| `completed_at` | timestamptz | yes | completion time |
| `cancelled_at` | timestamptz | yes | cancellation time |

### 4.21 `warehouse_exceptions`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `id` | uuid | no | exception ID |
| `tenant_id` | uuid | no | owning tenant |
| `order_id` | uuid | no | affected order |
| `task_type` | varchar(24) | no | picking or packing |
| `task_id` | uuid | no | affected task |
| `exception_type` | varchar(48) | no | missing_stock, damaged_stock, wrong_sku, partial_quantity, package_mismatch |
| `status` | varchar(32) | no | open, under_review, resolved, cancelled |
| `details` | jsonb | no | structured evidence |
| `resolution` | varchar(48) | yes | continue, adjust_inventory, remove_item, partial_fulfill, cancel, escalate |
| `resolved_by` | uuid | yes | resolving user |
| `created_at` | timestamptz | no | creation time |
| `resolved_at` | timestamptz | yes | resolution time |

### 4.22 `shipments`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `id` | uuid | no | shipment ID |
| `tenant_id` | uuid | no | owning tenant |
| `order_id` | uuid | no | unique order reference for one-package MVP |
| `provider` | varchar(32) | no | biteship for MVP |
| `provider_order_id` | varchar(128) | yes | provider shipment/order ID |
| `courier_code` | varchar(64) | yes | selected courier |
| `service_code` | varchar(64) | yes | selected service |
| `tracking_number` | varchar(128) | yes | tracking ID |
| `label_url` | text | yes | private or signed label reference |
| `shipping_cost_minor` | bigint | no | shipping amount |
| `status` | varchar(48) | no | canonical shipment state |
| `idempotency_key` | varchar(128) | no | tenant-unique shipment creation key |
| `version` | integer | no | concurrency version |
| `created_at` | timestamptz | no | creation time |
| `updated_at` | timestamptz | no | update time |
| `delivered_at` | timestamptz | yes | delivery time |
| `cancelled_at` | timestamptz | yes | cancellation time |

### 4.23 `shipment_tracking_events`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `id` | uuid | no | tracking event ID |
| `shipment_id` | uuid | no | parent shipment |
| `provider_event_id` | varchar(128) | yes | provider event ID |
| `provider_status` | varchar(128) | yes | original provider state |
| `canonical_status` | varchar(48) | no | normalized shipment state |
| `event_time` | timestamptz | yes | provider event time |
| `received_at` | timestamptz | no | platform receipt time |
| `location` | jsonb | yes | provider location data |
| `description` | text | yes | human-readable tracking text |
| `raw_reference` | jsonb | yes | sanitized provider reference |

### 4.24 `return_requests`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `id` | uuid | no | return ID |
| `tenant_id` | uuid | no | owning tenant |
| `order_id` | uuid | no | order |
| `status` | varchar(32) | no | return state |
| `reason_code` | varchar(64) | no | normalized reason |
| `customer_notes` | text | yes | customer explanation |
| `reviewed_by` | uuid | yes | reviewer |
| `version` | integer | no | concurrency version |
| `created_at` | timestamptz | no | creation time |
| `updated_at` | timestamptz | no | update time |
| `completed_at` | timestamptz | yes | completion time |

### 4.25 `refund_requests`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `id` | uuid | no | refund ID |
| `tenant_id` | uuid | no | owning tenant |
| `order_id` | uuid | no | order |
| `return_request_id` | uuid | yes | related return |
| `status` | varchar(32) | no | refund state |
| `requested_minor` | bigint | no | requested amount |
| `completed_minor` | bigint | no | completed amount |
| `currency` | char(3) | no | refund currency |
| `provider_reference` | varchar(128) | yes | provider refund reference |
| `idempotency_key` | varchar(128) | no | tenant-unique refund key |
| `version` | integer | no | concurrency version |
| `created_at` | timestamptz | no | creation time |
| `updated_at` | timestamptz | no | update time |
| `completed_at` | timestamptz | yes | completion time |

### 4.26 `provider_webhook_events`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `id` | uuid | no | event record ID |
| `tenant_id` | uuid | yes | resolved tenant, null before resolution |
| `provider` | varchar(32) | no | provider key |
| `provider_event_id` | varchar(160) | no | provider-unique event ID |
| `event_type` | varchar(128) | yes | provider event type |
| `payload_hash` | varchar(128) | no | duplicate and integrity helper |
| `status` | varchar(32) | no | received, processing, processed, failed, quarantined |
| `received_at` | timestamptz | no | receipt time |
| `processed_at` | timestamptz | yes | successful processing time |
| `failure_reason` | text | yes | sanitized failure information |

### 4.27 `idempotency_records`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `id` | uuid | no | record ID |
| `tenant_id` | uuid | yes | tenant or null for global operation |
| `scope` | varchar(64) | no | checkout, shipment, refund, webhook, etc. |
| `idempotency_key` | varchar(160) | no | request identity |
| `request_hash` | varchar(128) | no | prevents key reuse with different payload |
| `response_status` | integer | yes | persisted HTTP or operation status |
| `response_body` | jsonb | yes | replayable normalized result |
| `resource_type` | varchar(64) | yes | created or affected resource type |
| `resource_id` | varchar(128) | yes | created or affected resource ID |
| `expires_at` | timestamptz | yes | retention expiry |
| `created_at` | timestamptz | no | creation time |
| `completed_at` | timestamptz | yes | completion time |

### 4.28 `outbox_events`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `id` | uuid | no | event ID |
| `tenant_id` | uuid | yes | tenant or global |
| `aggregate_type` | varchar(64) | no | order, shipment, tenant, etc. |
| `aggregate_id` | varchar(128) | no | aggregate ID |
| `event_type` | varchar(128) | no | normalized domain event |
| `schema_version` | integer | no | payload contract version |
| `payload` | jsonb | no | event payload |
| `correlation_id` | uuid | yes | trace ID |
| `status` | varchar(24) | no | pending, publishing, published, failed |
| `attempt_count` | integer | no | publish attempts |
| `available_at` | timestamptz | no | next eligible publish time |
| `published_at` | timestamptz | yes | successful publication time |
| `created_at` | timestamptz | no | insertion time in same business transaction |

### 4.29 `notifications`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `id` | uuid | no | notification ID |
| `tenant_id` | uuid | no | tenant |
| `recipient_user_id` | uuid | yes | tenant user recipient |
| `recipient_customer_id` | uuid | yes | customer recipient |
| `type` | varchar(64) | no | notification type |
| `title` | varchar(220) | no | title |
| `body` | text | no | body |
| `data` | jsonb | no | resource references and UI metadata |
| `read_at` | timestamptz | yes | read time |
| `created_at` | timestamptz | no | creation time |

Exactly one recipient column must be non-null.

### 4.30 `audit_logs`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `id` | uuid | no | audit ID |
| `tenant_id` | uuid | yes | affected tenant |
| `actor_user_id` | uuid | yes | real actor |
| `effective_tenant_id` | uuid | yes | support-mode tenant context |
| `action` | varchar(128) | no | action key |
| `resource_type` | varchar(64) | no | resource category |
| `resource_id` | varchar(128) | yes | resource ID |
| `before_data` | jsonb | yes | sanitized prior values |
| `after_data` | jsonb | yes | sanitized new values |
| `ip_address` | inet | yes | actor IP |
| `user_agent` | text | yes | actor user agent |
| `correlation_id` | uuid | yes | trace ID |
| `created_at` | timestamptz | no | immutable event time |

### 4.31 `state_transitions`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `id` | uuid | no | transition ID |
| `tenant_id` | uuid | yes | owning tenant |
| `aggregate_type` | varchar(64) | no | tenant, order, payment, shipment, etc. |
| `aggregate_id` | varchar(128) | no | aggregate ID |
| `version_before` | integer | no | expected prior version |
| `version_after` | integer | no | resulting version |
| `state_before` | varchar(64) | no | prior state |
| `state_after` | varchar(64) | no | resulting state |
| `actor_user_id` | uuid | yes | actor or null for system |
| `reason` | text | yes | transition reason |
| `idempotency_key` | varchar(160) | yes | operation key |
| `correlation_id` | uuid | yes | trace ID |
| `created_at` | timestamptz | no | immutable time |

## 5. Ownership Path Reference

| Child Table | Tenant Ownership Path |
|---|---|
| `order_notes` | `order_notes.order_id -> order_extensions.tenant_id` |
| `receiving_lines` | `receiving_lines.receiving_id -> receiving_records.tenant_id` |
| `picking_lines` | `picking_lines.picking_task_id -> picking_tasks.tenant_id` |
| `shipment_tracking_events` | `shipment_tracking_events.shipment_id -> shipments.tenant_id` |
| `return_items` | `return_items.return_request_id -> return_requests.tenant_id` |

## 6. Sensitive Data Classification

| Classification | Examples | Rule |
|---|---|---|
| Public | product title, public image URL | may be cached publicly when tenant scoped |
| Internal | inventory, picking tasks, plan usage | authenticated tenant access only |
| Personal | name, email, addresses, IP | minimize, restrict, retain by policy |
| Secret | API keys, password hashes | never log or expose; encrypt or externalize |
| Audit-sensitive | before/after values, support mode | restricted access and append-only handling |

## 7. Required Validation

- Email normalization before uniqueness checks.
- Currency must be uppercase ISO code.
- Timezone must be valid IANA identifier.
- Quantities and dimensions must be non-negative; required package dimensions must be positive.
- Money values cannot be negative unless a documented accounting field explicitly permits signed values.
- State values must match the final state-machine specification.
- Every tenant-owned FK must resolve within the same tenant through application and database checks where feasible.

## 8. Change Summary

Initial data dictionary synchronized with the detailed database design and final lifecycle definitions.
