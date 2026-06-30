# Data Dictionary — Terra Commerce

**Document ID:** DATA-TC-002  
**Version:** 1.0.0  
**Status:** Final  
**Owner:** Technical Lead and Backend Lead  
**Last Updated:** 2026-06-30  
**Upstream:** Database Design v1.0.0 Final, State Machine v1.0.0 Final  
**Audit:** `reviews/Database_Design_Data_Dictionary_Audit_v1.0.0.md`

## 1. Common Conventions

| Name | Type | Meaning |
|---|---|---|
| `id` | uuid | primary identifier |
| `tenant_id` | uuid | owning tenant |
| `version` | integer | optimistic concurrency version starting at 1 |
| `created_at` | timestamptz | UTC creation time |
| `updated_at` | timestamptz | UTC modification time |
| `correlation_id` | uuid | cross-request trace ID |
| `idempotency_key` | varchar | stable operation identity |
| `*_minor` | bigint | money in minor units |
| `currency` | char(3) | uppercase ISO 4217 code |

Tenant-owned parents expose `UNIQUE (tenant_id, id)`. Tenant-owned children use composite tenant-safe foreign keys where practical.

## 2. Canonical Status Domains

- Tenant: `draft`, `trial`, `active`, `past_due`, `suspended`, `deactivated`, `archived`
- Order: `draft`, `pending_payment`, `payment_pending_verification`, `paid`, `processing`, `ready_for_fulfillment`, `picking`, `packing`, `ready_for_shipment`, `shipped`, `delivered`, `completed`, `cancellation_requested`, `cancelled`, `return_requested`, `returned`, `refund_pending`, `refunded`, `failed`
- Payment: `not_started`, `pending`, `pending_verification`, `authorized`, `captured`, `failed`, `cancelled`, `refund_pending`, `partially_refunded`, `refunded`
- Reservation: `pending`, `active`, `released`, `consumed`, `expired`, `failed`
- Fulfillment: `not_ready`, `ready`, `in_progress`, `exception`, `completed`, `cancelled`
- Picking: `not_created`, `ready`, `in_progress`, `paused`, `exception`, `completed`, `cancelled`
- Packing: `not_created`, `ready`, `in_progress`, `exception`, `completed`, `cancelled`
- Shipment: `shipment_pending`, `courier_assigned`, `awaiting_pickup`, `picked_up`, `in_transit`, `out_for_delivery`, `delivered`, `delivery_failed`, `returned_to_sender`, `cancelled`, `unknown_pending_review`
- Return: `requested`, `under_review`, `approved`, `rejected`, `in_transit`, `received`, `inspected`, `completed`, `cancelled`
- Refund: `not_required`, `requested`, `approved`, `rejected`, `processing`, `partially_completed`, `completed`, `failed`
- Job: `queued`, `running`, `retry_scheduled`, `completed`, `failed`, `dead_lettered`, `cancelled`

## 3. Global and Tenant Tables

### `plans`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `id` | uuid | no | primary key |
| `code` | varchar(64) | no | globally unique plan code |
| `name` | varchar(128) | no | display name |
| `status` | varchar(32) | no | active, inactive, archived |
| `trial_days` | integer | no | non-negative trial duration |
| `limits` | jsonb | no | resource limits by metric |
| `enforcement_policy` | jsonb | no | soft/hard policy by metric |
| timestamps | timestamptz | varies | lifecycle timestamps |

### `tenants`

| Column | Type | Null | Meaning |
|---|---|---:|---|
| `id` | uuid | no | primary key |
| `plan_id` | uuid | no | plan FK |
| `slug` | varchar(100) | no | global unique slug |
| `display_name` | varchar(180) | no | tenant name |
| `legal_name` | varchar(220) | yes | registered name |
| `status` | varchar(32) | no | tenant status |
| `timezone` | varchar(64) | no | IANA timezone |
| `currency` | char(3) | no | default currency |
| `version` | integer | no | concurrency version |
| lifecycle timestamps | timestamptz | varies | created, updated, suspended, deactivated, archived |

### `stores`

One row per tenant. Contains store name, description, contact details, address, branding JSON, settings JSON, and timestamps.

### `warehouses`

One row per tenant. Contains name, status, structured address, contact, Biteship origin area, operating hours, version, and timestamps.

### `users`

Global identity table with normalized unique email, password hash or external identity reference, status, last login, and timestamps.

### `tenant_memberships`

| Column | Meaning |
|---|---|
| `tenant_id` | owning tenant |
| `user_id` | global user |
| `status` | invited, active, deactivated, revoked |
| `is_owner` | tenant owner flag |
| invitation fields | inviter and lifecycle timestamps |

Unique `(tenant_id, user_id)`.

### Authorization tables

- `roles`: role key, name, scope, system flag
- `permissions`: unique action key and description
- `role_permissions`: unique role-permission relation
- `tenant_role_assignments`: tenant-safe membership-role relation

### `tenant_feature_overrides`

Tenant and feature composite key with enabled flag and effective dates.

### `tenant_usage_counters`

Tenant, metric, period, current value, and version. Unique by tenant, metric, and period start.

## 4. Medusa Extension Tables

### `product_tenant_links`

| Column | Meaning |
|---|---|
| `tenant_id` | product owner |
| `medusa_product_id` | stable Medusa reference |
| `status` | draft, active, inactive, archived |

Primary key `(tenant_id, medusa_product_id)`.

### `variant_tenant_links`

Tenant, Medusa variant reference, tenant-unique SKU, positive shipping weight and dimensions, timestamps.

### `customer_tenant_profiles`

Tenant, Medusa customer reference, tenant-unique normalized email, customer status, timestamps.

## 5. Order, Payment, and Fulfillment

### `order_extensions`

| Column | Type | Meaning |
|---|---|---|
| `tenant_id` | uuid | owning tenant |
| `medusa_order_id` | text | Medusa order reference |
| `order_number` | varchar(64) | tenant-unique human ID |
| status projections | varchar | business, payment, fulfillment, delivery read projections |
| `customer_id` | uuid | tenant customer profile |
| money fields | bigint | subtotal, discount, tax, shipping, total |
| address snapshots | jsonb | immutable order addresses |
| `pricing_snapshot` | jsonb | immutable item/adjustment snapshot |
| `version` | integer | order aggregate version |

### `payment_state_extensions`

| Column | Meaning |
|---|---|
| `tenant_id`, `order_id` | tenant-safe one-to-one order relation |
| `medusa_payment_reference` | supported Medusa/payment reference |
| `status` | payment state-machine value |
| amount fields | authorized, captured, refunded minor units |
| `currency` | immutable currency |
| `version` | payment aggregate version |

Checks: non-negative amounts; refunded <= captured.

### `fulfillment_state_extensions`

Tenant-safe one-to-one order relation, optional Medusa fulfillment reference, final fulfillment status, version, timestamps.

### `order_notes`

Indirectly owned through order. Stores author, internal body, visibility, and creation time.

## 6. Inventory and WMS Dictionary

### Inventory bucket definitions

| Bucket | Meaning |
|---|---|
| `physical_qty` | saleable on-hand stock; excludes damaged stock |
| `reserved_qty` | stock held by Active reservations |
| `available_qty` | computed `physical_qty - reserved_qty` |
| `incoming_qty` | expected but not accepted inbound stock |
| `damaged_qty` | physically present but unavailable stock |

### `inventory_items`

Tenant, warehouse, variant, SKU, all quantity buckets, low-stock threshold, version, timestamps.

Checks: all values non-negative; reserved <= physical.

### `inventory_reservations`

Tenant-safe references to inventory item and order, positive quantity, reservation state, expiry, tenant-unique idempotency key, version, and lifecycle timestamps.

### `inventory_movements`

Append-only inventory ledger.

| Column | Meaning |
|---|---|
| `quantity_bucket` | physical, reserved, incoming, damaged |
| `movement_type` | initial, receiving, manual_add, manual_deduct, reservation, release, fulfillment, return, damage, correction |
| before/delta/after | exact bucket arithmetic |
| reference fields | related order, reservation, receiving, return, or manual action |
| actor/reason/correlation | audit context |

### `receiving_records`

Tenant, warehouse, receiving status, reference, receiving actor, version, timestamps.

### `receiving_lines`

Parent receiving record, inventory item, expected, accepted, damaged, and rejected quantities.

Receiving effects:

- accepted: incoming decreases where applicable; physical increases
- damaged: incoming decreases where applicable; damaged increases
- rejected: incoming decreases where applicable; no sellable bucket increases

### `picking_tasks` and `picking_lines`

One task per order. Task stores tenant, order, warehouse, state, assignee, version, and timestamps. Lines store required and picked quantity plus line state.

### `packing_tasks`

One task per order. Stores state, assignee, positive final package weight and dimensions, version, and timestamps.

### `warehouse_exceptions`

Stores tenant, order, task type/reference, exception type, status, structured details, resolution, resolver, and timestamps.

## 7. Shipment, Return, and Refund

### `shipments`

Tenant-safe one-to-one order relation for MVP. Stores provider, provider order ID, courier/service, tracking number, label reference, shipping cost, canonical state, idempotency key, version, and timestamps.

### `shipment_tracking_events`

Append-only tracking history with provider event ID/status, canonical status, provider time, receipt time, location, description, and sanitized raw reference.

### `return_requests`

Tenant-safe order relation, return state, reason, customer notes, reviewer, version, and timestamps.

### `return_items`

Indirect tenant ownership through return request. Stores order item reference, positive quantity, and disposition.

### `refund_requests`

Tenant-safe order and optional return relation, refund state, requested/completed amounts, currency, provider reference, idempotency key, version, timestamps.

Checks: completed <= requested and totals must remain within captured payment.

## 8. Integration and Reliability

### `provider_webhook_events`

Provider, provider event ID, optional resolved tenant, event type, payload hash, processing status, timestamps, sanitized failure reason.

Unique `(provider, provider_event_id)`.

### `idempotency_records`

Tenant or global operation scope, idempotency key, request hash, replayable response, resource reference, expiry, timestamps.

Uniqueness:

- tenant partial unique: `(tenant_id, scope, idempotency_key)` where tenant is not null
- global partial unique: `(scope, idempotency_key)` where tenant is null

### `outbox_events`

| Column | Meaning |
|---|---|
| tenant and aggregate fields | event ownership and aggregate identity |
| `aggregate_version` | state version producing the event |
| `event_type`, `schema_version` | event contract identity |
| `payload` | normalized event payload |
| status and attempt fields | publishing lifecycle |
| claim metadata | worker claim owner and time |
| timestamps | availability, creation, publication |

Unique `(aggregate_type, aggregate_id, aggregate_version, event_type)`.

### Job persistence

`background_jobs` and `dead_letter_jobs` may persist durable operational metadata beyond Redis Streams, including tenant, type, status, idempotency identity, attempts, schedule, payload reference, and errors.

## 9. Notification, Report, Audit, and Transition Tables

### `notifications`

Tenant, exactly one user/customer recipient, type, title, body, structured data, read time, creation time.

XOR recipient constraint is mandatory.

### `generated_reports`

Tenant/global scope, requester, type, parameters, state, object key, expiry, timestamps.

### `audit_logs`

Append-only fields: tenant, actor, effective tenant, action, resource, sanitized before/after values, IP, user agent, correlation ID, creation time.

### `state_transitions`

Append-only fields: tenant, aggregate type/ID, versions before/after, states before/after, actor, reason, idempotency key, correlation ID, timestamp.

## 10. Ownership Paths for Indirect Children

| Child | Ownership Path |
|---|---|
| `order_notes` | order -> tenant |
| `receiving_lines` | receiving record -> tenant |
| `picking_lines` | picking task -> tenant |
| `shipment_tracking_events` | shipment -> tenant |
| `return_items` | return request -> tenant |

## 11. Database-Level Integrity Rules

1. Composite tenant-safe foreign keys where practical.
2. Exactly one notification recipient.
3. Positive package dimensions before packing completion.
4. Non-negative inventory buckets and money amounts.
5. Reserved inventory never exceeds physical inventory.
6. Refunded amounts never exceed captured payment.
7. Append-only tables deny normal update/delete.
8. Stateful aggregate updates compare `version` atomically.
9. Outbox event is inserted in the same transaction as its state change.
10. A reused idempotency key with a different request hash is rejected.

## 12. Sensitive Data Classes

| Class | Examples | Handling |
|---|---|---|
| Public | product title, public media | tenant-scoped public caching allowed |
| Internal | inventory, task queues, usage | authenticated tenant access |
| Personal | email, address, IP | minimize and restrict |
| Secret | passwords, API keys | never log; encrypt or externalize |
| Audit-sensitive | support mode, before/after | restricted append-only access |

## 13. Validation Rules

- Normalize email before uniqueness checks.
- Validate IANA timezone and ISO currency.
- Require positive dimensions when shipping/packing needs them.
- Require reasons for manual and correction movements.
- Validate state values against the final state machine.
- Verify all tenant-owned references belong to the same tenant.
- Validate JSON structures at application boundary and optionally with database checks for stable fields.

## Change Summary

Final dictionary incorporates tenant-safe composite relationships, explicit payment and fulfillment persistence, precise inventory bucket semantics, null-safe idempotency uniqueness, aggregate-version outbox ordering, append-only enforcement, and notification XOR constraints.
