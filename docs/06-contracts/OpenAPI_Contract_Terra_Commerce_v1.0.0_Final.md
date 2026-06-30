# OpenAPI Contract — Terra Commerce

**Document ID:** API-TC-001  
**Version:** 1.0.0  
**Status:** Final  
**Owner:** Backend Lead  
**Last Updated:** 2026-06-30  
**Upstream:** FSD v1.0.1, System Architecture v1.0.0 Final, State Machine v1.0.0 Final, Database Design v1.0.0 Final  
**Audit:** `reviews/Contract_Suite_Audit_v1.0.0.md`

## 1. Scope and Base Rules

Public base path: `/api/v1`.

All public clients use the Go gateway. JSON is the default representation. `X-Correlation-ID` is accepted and returned. Money uses minor units and ISO currency. Times use RFC 3339 UTC.

Tenant context is resolved and validated server-side. Request-body tenant IDs are never authoritative.

## 2. Command Metadata

Every machine-readable OpenAPI operation must define:

- stable `operationId`
- audience: platform user, tenant user, customer, or provider
- permission key where applicable
- ownership rule
- authentication scheme
- idempotency requirement
- expected aggregate version or `If-Match` requirement
- allowed source states and result state for lifecycle commands
- emitted domain events
- audit requirement
- documented errors and examples

## 3. Common Headers

- `Authorization` or approved secure session mechanism
- `X-Correlation-ID`
- `Idempotency-Key` for checkout, payment confirmation, cancellation approval, shipment creation/cancellation, refund creation/retry, and operational retries
- `If-Match` for versioned mutable resources and state commands
- `Accept: text/event-stream` for SSE

## 4. Standard Envelopes

Success:

```json
{"data":{},"meta":{"correlation_id":"uuid","version":1}}
```

Error:

```json
{"error":{"code":"invalid_state_transition","message":"...","details":{},"correlation_id":"uuid"}}
```

Canonical errors: `validation_error`, `authentication_required`, `forbidden`, `resource_not_found`, `state_conflict`, `invalid_state_transition`, `plan_limit_exceeded`, `rate_limit_exceeded`, `provider_unavailable`, `maintenance_mode`, `internal_error`.

## 5. Endpoint Inventory

### Platform

- `POST /platform/tenants`
- `GET /platform/tenants`
- `GET /platform/tenants/{tenant_id}`
- `PATCH /platform/tenants/{tenant_id}`
- `POST /platform/tenants/{tenant_id}/status-transitions`
- `POST /platform/tenants/{tenant_id}/owner-access-resets`
- `POST /platform/tenants/{tenant_id}/support-sessions`
- `POST /platform/plans`
- `GET /platform/plans`
- `GET /platform/plans/{plan_id}`
- `PATCH /platform/plans/{plan_id}`
- `POST /platform/plans/{plan_id}/archive-requests`
- `POST /platform/tenants/{tenant_id}/plan-transitions`
- `GET /platform/tenants/{tenant_id}/usage`
- `GET /platform/health`
- `GET /platform/jobs/failed`
- `POST /platform/jobs/{job_id}/retry-requests`
- `GET /platform/integrations/failures`
- `PATCH /platform/settings/{key}`
- `PATCH /platform/feature-flags/{feature_key}`
- `POST /platform/maintenance-sessions`
- `DELETE /platform/maintenance-sessions/{session_id}`

### Tenant administration

- `GET /tenant/profile`
- `PATCH /tenant/profile`
- `GET /tenant/store`
- `PATCH /tenant/store`
- `GET /tenant/warehouse`
- `PATCH /tenant/warehouse`
- `GET /tenant/users`
- `POST /tenant/user-invitations`
- `POST /tenant/user-invitations/{invitation_id}/resend-requests`
- `DELETE /tenant/user-invitations/{invitation_id}`
- `PATCH /tenant/users/{membership_id}`
- `GET /tenant/roles`
- `PUT /tenant/users/{membership_id}/roles`
- `GET /tenant/usage`

### Catalog

- `POST /catalog/products`
- `GET /catalog/products`
- `GET /catalog/products/{product_id}`
- `PATCH /catalog/products/{product_id}`
- `POST /catalog/products/{product_id}/publish-requests`
- `POST /catalog/products/{product_id}/deactivation-requests`
- `POST /catalog/products/{product_id}/archive-requests`
- `POST /catalog/products/{product_id}/variants`
- `PATCH /catalog/variants/{variant_id}`
- `POST /catalog/categories`
- `PATCH /catalog/categories/{category_id}`
- `POST /catalog/products/{product_id}/images`
- `DELETE /catalog/products/{product_id}/images/{image_id}`

### Storefront and customer

- `POST /storefront/customers/register`
- `POST /storefront/customers/login`
- `POST /storefront/customers/logout`
- `POST /storefront/customers/password-reset-requests`
- `POST /storefront/customers/password-reset-confirmations`
- `GET /storefront/customers/me`
- `GET|POST /storefront/customers/me/addresses`
- `PATCH|DELETE /storefront/customers/me/addresses/{address_id}`
- `GET /storefront/products`
- `GET /storefront/products/{product_id}`
- `GET /storefront/categories`
- `POST /storefront/carts`
- `GET /storefront/carts/{cart_id}`
- `POST /storefront/carts/{cart_id}/items`
- `PATCH|DELETE /storefront/carts/{cart_id}/items/{item_id}`
- `POST /storefront/carts/{cart_id}/claim-requests`
- `POST /storefront/carts/{cart_id}/shipping-rate-requests`
- `POST /storefront/carts/{cart_id}/checkout-requests`
- `GET /storefront/orders`
- `GET /storefront/orders/{order_id}`
- `GET /storefront/orders/{order_id}/tracking`
- `POST /storefront/orders/{order_id}/cancellation-requests`
- `POST /storefront/orders/{order_id}/return-requests`

### OMS

- `GET /orders`
- `GET /orders/{order_id}`
- `POST /orders/{order_id}/payment-confirmations`
- `POST /orders/{order_id}/cancellation-requests`
- `POST /orders/{order_id}/cancellations/{request_id}/approval-requests`
- `POST /orders/{order_id}/cancellations/{request_id}/rejection-requests`
- `POST /orders/{order_id}/notes`
- `GET /orders/{order_id}/timeline`
- `GET /orders/{order_id}/returns`
- `POST /orders/{order_id}/returns/{return_id}/approval-requests`
- `POST /orders/{order_id}/returns/{return_id}/rejection-requests`
- `POST /orders/{order_id}/refunds`
- `POST /orders/{order_id}/refunds/{refund_id}/retry-requests`

### Inventory and WMS

- `GET /inventory/items`
- `GET /inventory/items/{inventory_item_id}`
- `GET /inventory/movements`
- `POST /inventory/adjustments`
- `GET /inventory/alerts`
- `POST /warehouse/receivings`
- `GET /warehouse/receivings`
- `GET|PATCH /warehouse/receivings/{receiving_id}`
- `POST /warehouse/receivings/{receiving_id}/completion-requests`
- `POST /warehouse/receivings/{receiving_id}/cancellation-requests`
- `GET /warehouse/picking-tasks`
- `GET /warehouse/picking-tasks/{task_id}`
- `POST /warehouse/picking-tasks/{task_id}/start-requests`
- `POST /warehouse/picking-tasks/{task_id}/pause-requests`
- `POST /warehouse/picking-tasks/{task_id}/resume-requests`
- `POST /warehouse/picking-tasks/{task_id}/exceptions`
- `POST /warehouse/picking-tasks/{task_id}/completion-requests`
- `GET /warehouse/picking-tasks/{task_id}/printable`
- `GET /warehouse/packing-tasks`
- `GET /warehouse/packing-tasks/{task_id}`
- `POST /warehouse/packing-tasks/{task_id}/start-requests`
- `POST /warehouse/packing-tasks/{task_id}/exceptions`
- `POST /warehouse/packing-tasks/{task_id}/completion-requests`

### Shipments

- `POST /orders/{order_id}/shipments`
- `GET /orders/{order_id}/shipments`
- `GET /shipments/{shipment_id}`
- `GET /shipments/{shipment_id}/tracking`
- `GET /shipments/{shipment_id}/label`
- `POST /shipments/{shipment_id}/cancellation-requests`
- `POST /shipments/{shipment_id}/reconciliation-requests`

### Reporting, notification, audit, SSE, webhook

- `GET /dashboard`
- `POST /reports`
- `GET /reports`
- `GET /reports/{report_id}`
- `GET /reports/{report_id}/download`
- `GET /notifications`
- `POST /notifications/{notification_id}/read-receipts`
- `GET /audit-logs`
- `GET /events/stream`
- `POST /webhooks/biteship`

## 6. Naming Rule

Generic lifecycle transition endpoints are permitted only for global administrative lifecycles such as tenant status. Domain commands use explicit action resources such as `completion-requests`, `approval-requests`, or `cancellation-requests`.

## 7. Machine-Readable Artifact

A normative `openapi.yaml` must be generated before implementation. It must include full component schemas, security schemes, operation IDs, examples, permissions, state metadata, pagination, error responses, and webhook/SSE documentation.

## 8. Acceptance Criteria

- Every FSD command is represented or marked internal-only.
- Every state command matches the final state machine.
- Command metadata is complete.
- Tenant and ownership rules are explicit.
- Errors, idempotency, and concurrency are normalized.

## Change Summary

Final version closes CONTRACT-M01 and CONTRACT-M02 by requiring stable operation metadata and normalized action-resource naming.
