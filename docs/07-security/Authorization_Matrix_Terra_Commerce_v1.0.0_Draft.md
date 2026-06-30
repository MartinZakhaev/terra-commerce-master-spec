# Authorization Matrix — Terra Commerce

**Document ID:** SEC-TC-001  
**Version:** 1.0.0  
**Status:** Draft  
**Owner:** Security Owner and Backend Lead  
**Last Updated:** 2026-06-30  
**Upstream:** FSD v1.0.1, System Architecture v1.0.0 Final, State Machine v1.0.0 Final, OpenAPI Contract v1.0.0 Final, Data Dictionary v1.0.0 Final

## 1. Purpose

This document defines who may perform each protected action, under which tenant, ownership, state, plan, and support-mode conditions.

## 2. Authorization Model

Authorization is evaluated in this order:

1. Identity is authenticated.
2. User or customer status is active.
3. Tenant is resolved and active for the requested operation.
4. Actor is a valid tenant member or global platform user.
5. Required permission is present.
6. Resource belongs to the resolved tenant.
7. Resource ownership is satisfied for customer-owned resources.
8. Current state permits the action.
9. Plan and feature flags permit the capability.
10. Support-mode or emergency restrictions are satisfied.

UI visibility is advisory only. The gateway and downstream domain module must enforce the same effective rule.

## 3. Roles

### Global roles

- Global Superadmin
- Global Operations
- Global Support
- Global Auditor

### Tenant roles

- Tenant Owner
- Tenant Administrator
- Order Manager
- Warehouse Manager
- Warehouse Staff
- Customer Support
- Finance Viewer
- Read-Only Auditor

### External identities

- Customer
- Biteship Provider
- System Worker

## 4. Permission Naming

Permissions use `resource.action` notation.

Examples:

- `tenant.create`
- `tenant.update`
- `order.read`
- `order.cancel`
- `inventory.adjust`
- `shipment.create`
- `audit.read`

## 5. Global Platform Matrix

Legend: `A` allowed, `R` read-only, `C` conditional, `-` denied.

| Capability | Superadmin | Operations | Support | Global Auditor |
|---|---:|---:|---:|---:|
| Create tenant | A | - | - | - |
| Read tenant list/detail | A | A | A | R |
| Edit tenant profile | A | C | - | - |
| Change tenant status | A | C | - | - |
| Reset owner access | A | - | C | - |
| Start support session | A | - | A | - |
| Manage plans | A | - | - | R |
| Change tenant plan | A | - | - | R |
| Read usage | A | A | A | R |
| Manage global settings | A | - | - | R |
| Manage feature flags | A | C | - | R |
| Enable maintenance mode | A | A | - | R |
| Read platform health | A | A | R | R |
| Retry failed jobs | A | A | - | R |
| Read integration failures | A | A | R | R |
| Read global audit logs | A | R | R | R |

Conditions:

- Operations may change tenant status only for operationally approved transitions and never archive tenants.
- Support may reset owner access only through audited recovery workflow.
- Support sessions are time-limited, explicitly confirmed, and always audited.

## 6. Tenant Administration Matrix

| Capability | Owner | Tenant Admin | Order Manager | Warehouse Manager | Warehouse Staff | Support | Finance | Auditor |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| Read tenant profile | A | A | R | R | R | R | R | R |
| Update tenant profile | A | A | - | - | - | - | - | - |
| Update store settings | A | A | - | - | - | - | - | - |
| Update warehouse settings | A | A | - | A | - | - | - | - |
| Invite tenant users | A | A | - | - | - | - | - | - |
| Deactivate tenant users | A | A | - | - | - | - | - | - |
| Assign tenant roles | A | A | - | - | - | - | - | R |
| Read plan usage | A | A | R | R | R | R | R | R |
| Read tenant audit logs | A | A | R | R | - | R | R | R |

Rules:

- A tenant must retain at least one active Tenant Owner.
- Tenant administrators cannot assign global roles.
- A user cannot assign a role that grants permissions above the assigner's effective permission ceiling.

## 7. Catalog Matrix

| Capability | Owner | Tenant Admin | Order Manager | Warehouse Manager | Warehouse Staff | Support | Finance | Auditor |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| Read catalog | A | A | R | R | R | R | R | R |
| Create product | A | A | - | - | - | - | - | - |
| Update product | A | A | - | - | - | - | - | - |
| Publish/deactivate/archive product | A | A | - | - | - | - | - | - |
| Manage variants | A | A | - | R | R | - | - | R |
| Manage categories | A | A | - | - | - | - | - | R |
| Manage product images | A | A | - | - | - | - | - | - |

Additional guards:

- Plan product limit.
- Tenant ownership.
- Publish validation.
- Historical variants are archived rather than destructively deleted.

## 8. Order and Payment Matrix

| Capability | Owner | Tenant Admin | Order Manager | Warehouse Manager | Warehouse Staff | Support | Finance | Auditor |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| List/read orders | A | A | A | R | R | A | R | R |
| Read customer contact snapshot | A | A | A | C | C | A | R | R |
| Add internal order note | A | A | A | R | - | A | - | R |
| Confirm manual payment | A | A | A | - | - | - | C | R |
| Request cancellation | A | A | A | C | C | A | - | R |
| Approve/reject cancellation | A | A | A | - | - | - | - | R |
| Approve/reject return | A | A | A | C | - | A | - | R |
| Create/refund retry | A | A | A | - | - | - | C | R |
| Read order timeline | A | A | A | A | A | A | R | R |

Conditions:

- Manual payment confirmation requires `order.payment_confirm` and a valid payment state.
- Finance may confirm or refund only when explicitly granted a tenant-specific elevated permission; default Finance Viewer is read-only.
- Warehouse roles see only customer data required for fulfillment.
- Support may not confirm payment or issue refunds.

## 9. Inventory and WMS Matrix

| Capability | Owner | Tenant Admin | Order Manager | Warehouse Manager | Warehouse Staff | Support | Finance | Auditor |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| Read inventory | A | A | R | A | A | R | R | R |
| Adjust inventory | A | A | - | A | C | - | - | R |
| Read movements | A | A | R | A | A | R | R | R |
| Create receiving | A | A | - | A | A | - | - | R |
| Complete/cancel receiving | A | A | - | A | C | - | - | R |
| Start/pause/resume picking | A | A | - | A | A | - | - | R |
| Report picking exception | A | A | C | A | A | A | - | R |
| Complete picking | A | A | - | A | A | - | - | R |
| Start/complete packing | A | A | - | A | A | - | - | R |
| Resolve warehouse exception | A | A | C | A | C | - | - | R |
| Print picking list/label | A | A | A | A | A | A | - | R |

Conditions:

- Warehouse Staff inventory adjustments may be limited to configured movement types and thresholds.
- Completion commands require current task assignment or manager override.
- All manual adjustments require reason and audit record.

## 10. Shipment Matrix

| Capability | Owner | Tenant Admin | Order Manager | Warehouse Manager | Warehouse Staff | Support | Finance | Auditor |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| Create shipment | A | A | A | A | C | - | - | R |
| Read shipment/tracking | A | A | A | A | A | A | R | R |
| Read/download label | A | A | A | A | A | A | - | R |
| Cancel shipment | A | A | A | C | - | - | - | R |
| Reconcile unknown shipment | A | A | A | C | - | C | - | R |

Conditions:

- Shipment creation requires payment, picking, packing, and package guards.
- Support reconciliation is recommendation-only unless explicitly granted operational override.
- Label access is tenant scoped and may contain personal data.

## 11. Reporting, Notification, and Audit Matrix

| Capability | Owner | Tenant Admin | Order Manager | Warehouse Manager | Warehouse Staff | Support | Finance | Auditor |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| Read dashboard | A | A | A | A | C | A | A | R |
| Create sales/order report | A | A | A | R | - | R | A | R |
| Create inventory/WMS report | A | A | R | A | C | R | R | R |
| Download generated report | A | A | C | C | - | C | A | R |
| Read notifications | own | own | own | own | own | own | own | own |
| Mark notification read | own | own | own | own | own | own | own | own |
| Read tenant audit logs | A | A | C | C | - | C | C | A |

## 12. Customer Matrix

| Capability | Customer Rule |
|---|---|
| Browse catalog | resolved tenant only |
| Manage cart | own anonymous or authenticated cart only |
| Claim cart | authenticated customer and valid claim token |
| Checkout | authenticated customer, own cart, active tenant |
| Manage addresses | own addresses only |
| Read orders | own orders only |
| Read payment status | own orders only |
| Track shipment | own order shipment only |
| Request cancellation | own eligible order only |
| Request return | own eligible delivered/completed order only |
| Receive SSE | own resources only |

## 13. Provider and System Matrix

### Biteship Provider

Allowed only:

- submit verified webhook payloads
- no general API access
- no tenant-selected authorization

### System Worker

Allowed only:

- consume approved event families
- execute scoped background jobs
- publish notifications/reports
- retry provider operations
- update resources through service identities with least privilege

Workers must carry tenant context and may not use unrestricted database superuser credentials.

## 14. Sensitive Field Rules

| Data | Default access |
|---|---|
| Password hashes and secrets | never returned |
| Customer email/phone | order/support roles with need-to-know |
| Full shipping address | order and fulfillment roles only |
| Audit before/after values | owner, tenant admin, auditor, limited support |
| Raw provider payload | operations/security only |
| Payment instrument details | never stored or returned beyond provider-safe metadata |
| Internal notes | tenant staff only; never customer-visible |

## 15. Support Mode Rules

- Requires `support.session.start`.
- Requires explicit tenant and reason.
- Has fixed expiry.
- Displays visible banner.
- Retains real actor and effective tenant.
- Cannot reveal secrets or password hashes.
- Cannot bypass state transitions.
- Mutating actions require separate elevated permission and are audited.

## 16. Endpoint-to-Permission Mapping Rules

Every OpenAPI operation must declare exactly one primary permission key or an explicit public/customer/provider rule.

Examples:

- `POST /platform/tenants` -> `tenant.create`
- `PATCH /tenant/store` -> `store.update`
- `POST /orders/{order_id}/payment-confirmations` -> `order.payment_confirm`
- `POST /inventory/adjustments` -> `inventory.adjust`
- `POST /orders/{order_id}/shipments` -> `shipment.create`
- `GET /audit-logs` -> `audit.read`

## 17. Deny Rules

The system must deny when:

- tenant context is missing or mismatched
- resource tenant differs from resolved tenant
- customer ownership fails
- membership is inactive
- tenant is suspended/deactivated for the requested mutation
- permission is absent
- state guard fails
- plan or feature is unavailable
- support session expired
- aggregate version is stale

## 18. Audit Requirements

Mandatory audit actions include:

- tenant lifecycle and plan changes
- support sessions
- user/role changes
- payment confirmation
- cancellation/return/refund decisions
- inventory adjustments and receiving completion
- shipment creation/cancellation/reconciliation
- feature flags, maintenance mode, and operational retries

## 19. Acceptance Criteria

1. Every protected OpenAPI operation has a permission or explicit audience rule.
2. Every state command includes state, ownership, and version guards.
3. Customer access is self-only.
4. Tenant roles cannot cross tenant boundaries.
5. Support mode cannot silently elevate privileges.
6. Sensitive fields follow least-privilege disclosure.
7. Denied actions are logged without leaking sensitive data.

## Change Summary

Initial detailed authorization matrix synchronized with the final contracts, state machines, architecture, and data model.
