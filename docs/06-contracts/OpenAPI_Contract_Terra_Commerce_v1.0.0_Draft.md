# OpenAPI Contract — Terra Commerce

**Document ID:** API-TC-001  
**Version:** 1.0.0  
**Status:** Draft  
**Owner:** Backend Lead  
**Last Updated:** 2026-06-30  
**Upstream:** FSD v1.0.1, System Architecture v1.0.0 Final, State Machine v1.0.0 Final, Database Design v1.0.0 Final

## 1. Contract Scope

This document defines the public HTTP API surface exposed by the Go gateway. Frontends must not call Medusa, PostgreSQL, or Redis directly.

Base path: `/api/v1`

## 2. Global Conventions

- JSON request and response bodies.
- `Content-Type: application/json` unless file upload/download applies.
- Bearer or session authentication as selected by future ADR.
- `X-Correlation-ID` accepted and returned.
- `Idempotency-Key` required for checkout, shipment creation, payment confirmation, cancellation, refund, and retryable commands.
- `If-Match` or body `version` required for mutable state commands.
- Tenant context is resolved server-side and never trusted solely from request body.
- Pagination uses `page`, `limit`, `sort`, and `cursor` where appropriate.
- Times use RFC 3339 UTC.
- Money uses `{ amount_minor, currency }`.

## 3. Standard Response Envelope

Success:

```json
{
  "data": {},
  "meta": {
    "correlation_id": "uuid",
    "version": 1
  }
}
```

Error:

```json
{
  "error": {
    "code": "invalid_state_transition",
    "message": "Human-readable message",
    "details": {},
    "correlation_id": "uuid"
  }
}
```

Standard error codes:

- `validation_error`
- `authentication_required`
- `forbidden`
- `resource_not_found`
- `state_conflict`
- `invalid_state_transition`
- `plan_limit_exceeded`
- `rate_limit_exceeded`
- `provider_unavailable`
- `maintenance_mode`
- `internal_error`

## 4. Security Schemes

- `userAuth`: authenticated platform or tenant user.
- `customerAuth`: authenticated tenant customer.
- `webhookAuth`: provider signature verification.

Every operation declares required permission and ownership checks.

## 5. Superadmin and Platform Endpoints

### Tenants

- `POST /platform/tenants` — create tenant.
- `GET /platform/tenants` — list tenants.
- `GET /platform/tenants/{tenant_id}` — tenant detail.
- `PATCH /platform/tenants/{tenant_id}` — edit tenant.
- `POST /platform/tenants/{tenant_id}/transitions` — change tenant status.
- `POST /platform/tenants/{tenant_id}/owner-access/reset` — reset owner access.
- `POST /platform/tenants/{tenant_id}/support-sessions` — begin audited support mode.

### Plans and usage

- `POST /platform/plans`
- `GET /platform/plans`
- `GET /platform/plans/{plan_id}`
- `PATCH /platform/plans/{plan_id}`
- `POST /platform/plans/{plan_id}/archive`
- `POST /platform/tenants/{tenant_id}/plan-transitions`
- `GET /platform/tenants/{tenant_id}/usage`

### Platform operations

- `GET /platform/health`
- `GET /platform/jobs/failed`
- `POST /platform/jobs/{job_id}/retry`
- `GET /platform/integrations/failures`
- `PATCH /platform/settings/{key}`
- `PATCH /platform/feature-flags/{feature_key}`
- `POST /platform/maintenance-mode`
- `DELETE /platform/maintenance-mode`

## 6. Tenant Administration Endpoints

- `GET /tenant/profile`
- `PATCH /tenant/profile`
- `GET /tenant/store`
- `PATCH /tenant/store`
- `GET /tenant/warehouse`
- `PATCH /tenant/warehouse`
- `GET /tenant/users`
- `POST /tenant/users/invitations`
- `POST /tenant/users/invitations/{invitation_id}/resend`
- `DELETE /tenant/users/invitations/{invitation_id}`
- `PATCH /tenant/users/{membership_id}`
- `GET /tenant/roles`
- `PUT /tenant/users/{membership_id}/roles`
- `GET /tenant/usage`

## 7. Catalog Endpoints

- `POST /catalog/products`
- `GET /catalog/products`
- `GET /catalog/products/{product_id}`
- `PATCH /catalog/products/{product_id}`
- `POST /catalog/products/{product_id}/publish`
- `POST /catalog/products/{product_id}/deactivate`
- `POST /catalog/products/{product_id}/archive`
- `POST /catalog/products/{product_id}/variants`
- `PATCH /catalog/variants/{variant_id}`
- `POST /catalog/categories`
- `PATCH /catalog/categories/{category_id}`
- `POST /catalog/products/{product_id}/images`
- `DELETE /catalog/products/{product_id}/images/{image_id}`

## 8. Storefront and Customer Endpoints

### Customer identity

- `POST /storefront/customers/register`
- `POST /storefront/customers/login`
- `POST /storefront/customers/logout`
- `POST /storefront/customers/password-reset/request`
- `POST /storefront/customers/password-reset/confirm`
- `GET /storefront/customers/me`
- `GET /storefront/customers/me/addresses`
- `POST /storefront/customers/me/addresses`
- `PATCH /storefront/customers/me/addresses/{address_id}`
- `DELETE /storefront/customers/me/addresses/{address_id}`

### Public catalog

- `GET /storefront/products`
- `GET /storefront/products/{product_id}`
- `GET /storefront/categories`

### Cart and checkout

- `POST /storefront/carts`
- `GET /storefront/carts/{cart_id}`
- `POST /storefront/carts/{cart_id}/items`
- `PATCH /storefront/carts/{cart_id}/items/{item_id}`
- `DELETE /storefront/carts/{cart_id}/items/{item_id}`
- `POST /storefront/carts/{cart_id}/claim`
- `POST /storefront/carts/{cart_id}/shipping-rates`
- `POST /storefront/carts/{cart_id}/checkout`

### Customer orders

- `GET /storefront/orders`
- `GET /storefront/orders/{order_id}`
- `GET /storefront/orders/{order_id}/tracking`
- `POST /storefront/orders/{order_id}/cancellation-requests`
- `POST /storefront/orders/{order_id}/return-requests`

## 9. OMS Endpoints

- `GET /orders`
- `GET /orders/{order_id}`
- `POST /orders/{order_id}/payment-confirmations`
- `POST /orders/{order_id}/cancellation-requests`
- `POST /orders/{order_id}/cancellations/{request_id}/approve`
- `POST /orders/{order_id}/cancellations/{request_id}/reject`
- `POST /orders/{order_id}/notes`
- `GET /orders/{order_id}/timeline`
- `GET /orders/{order_id}/returns`
- `POST /orders/{order_id}/returns/{return_id}/approve`
- `POST /orders/{order_id}/returns/{return_id}/reject`
- `POST /orders/{order_id}/refunds`
- `POST /orders/{order_id}/refunds/{refund_id}/retry`

## 10. Inventory and WMS Endpoints

### Inventory

- `GET /inventory/items`
- `GET /inventory/items/{inventory_item_id}`
- `GET /inventory/movements`
- `POST /inventory/adjustments`
- `GET /inventory/alerts`

### Receiving

- `POST /warehouse/receivings`
- `GET /warehouse/receivings`
- `GET /warehouse/receivings/{receiving_id}`
- `PATCH /warehouse/receivings/{receiving_id}`
- `POST /warehouse/receivings/{receiving_id}/complete`
- `POST /warehouse/receivings/{receiving_id}/cancel`

### Picking and packing

- `GET /warehouse/picking-tasks`
- `GET /warehouse/picking-tasks/{task_id}`
- `POST /warehouse/picking-tasks/{task_id}/start`
- `POST /warehouse/picking-tasks/{task_id}/pause`
- `POST /warehouse/picking-tasks/{task_id}/resume`
- `POST /warehouse/picking-tasks/{task_id}/exceptions`
- `POST /warehouse/picking-tasks/{task_id}/complete`
- `GET /warehouse/picking-tasks/{task_id}/printable`
- `GET /warehouse/packing-tasks`
- `GET /warehouse/packing-tasks/{task_id}`
- `POST /warehouse/packing-tasks/{task_id}/start`
- `POST /warehouse/packing-tasks/{task_id}/exceptions`
- `POST /warehouse/packing-tasks/{task_id}/complete`

## 11. Shipment Endpoints

- `POST /orders/{order_id}/shipments`
- `GET /orders/{order_id}/shipments`
- `GET /shipments/{shipment_id}`
- `GET /shipments/{shipment_id}/tracking`
- `GET /shipments/{shipment_id}/label`
- `POST /shipments/{shipment_id}/cancel`
- `POST /shipments/{shipment_id}/reconcile`

## 12. Reporting, Notification, and Audit Endpoints

- `GET /dashboard`
- `POST /reports`
- `GET /reports`
- `GET /reports/{report_id}`
- `GET /reports/{report_id}/download`
- `GET /notifications`
- `POST /notifications/{notification_id}/read`
- `GET /audit-logs`

## 13. SSE Endpoint

- `GET /events/stream`

Headers:

- `Accept: text/event-stream`
- `Last-Event-ID` optional
- authenticated tenant/customer context required

## 14. Webhook Endpoint

- `POST /webhooks/biteship`

The endpoint uses provider signature verification, provider-event deduplication, normalized status mapping, and always returns a provider-compatible acknowledgement after safe persistence.

## 15. Command Requirements

State-changing operations declare:

- required permission
- expected aggregate version
- idempotency requirement
- allowed source states
- result state
- emitted domain events
- audit requirement

## 16. OpenAPI Delivery Requirement

A machine-readable `openapi.yaml` must be generated from this contract before implementation. Every endpoint must include schemas, examples, permissions, errors, pagination, and operation IDs.

## 17. Acceptance Criteria

1. Every FSD command has a public API operation or documented internal-only workflow.
2. Every state-changing operation maps to the final state machine.
3. Tenant context is never trusted from body alone.
4. Idempotency and concurrency requirements are explicit.
5. Error codes are normalized.
6. Frontends use gateway endpoints only.

## Change Summary

Initial OpenAPI contract draft synchronized with the final architecture, state machines, and data model.
