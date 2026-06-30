# Authorization Matrix — Terra Commerce

**Document ID:** SEC-TC-001  
**Version:** 1.0.0  
**Status:** Final  
**Owner:** Security Owner and Backend Lead  
**Last Updated:** 2026-06-30  
**Upstream:** FSD v1.0.1, System Architecture v1.0.0 Final, State Machine v1.0.0 Final, OpenAPI Contract v1.0.0 Final, Data Dictionary v1.0.0 Final  
**Audit:** `reviews/Authorization_Threat_Model_Audit_v1.0.0.md`

## 1. Authorization Evaluation Order

Every protected operation is denied unless all applicable checks pass:

1. Identity is authenticated.
2. Actor status is active.
3. Tenant is resolved and valid.
4. Membership or global role is valid.
5. Required permission is present.
6. Resource belongs to the resolved tenant.
7. Customer ownership is satisfied where applicable.
8. Current lifecycle state permits the command.
9. Aggregate version is current.
10. Plan and feature flags permit the capability.
11. Support-mode, dual-control, and emergency restrictions are satisfied.

The system is deny-by-default. UI visibility is never an authorization control.

## 2. Conditional Grant Rule

Every `C` grant is denied by default. It becomes allowed only when an explicit permission, configured threshold, assignment, task ownership, or approved policy condition is satisfied.

Key conditional permissions include:

- `finance.payment_confirm`
- `finance.refund_execute`
- `warehouse.inventory_adjust_limited`
- `warehouse.task_override`
- `support.owner_access_reset`
- `support.shipment_reconcile`
- `operations.tenant_status_transition`
- `operations.feature_flag_update`
- `support.mutate_in_session`

## 3. Roles

Global:

- Global Superadmin
- Global Operations
- Global Support
- Global Auditor

Tenant:

- Tenant Owner
- Tenant Administrator
- Order Manager
- Warehouse Manager
- Warehouse Staff
- Customer Support
- Finance Viewer
- Read-Only Auditor

External/system:

- Customer
- Biteship Provider
- System Worker

## 4. Permission Registry

Permissions use `resource.action` notation. Every OpenAPI `operationId` must map to exactly one primary permission or an explicit public/customer/provider rule.

A machine-checkable registry must contain:

- `operationId`
- HTTP method and path
- audience
- permission key
- ownership rule
- allowed source states
- support-mode policy
- audit requirement
- dual-control requirement

CI must fail when an operation has no mapping, multiple conflicting mappings, or references an unknown permission.

## 5. Global Platform Matrix

Legend: `A` allowed, `R` read-only, `C` conditional, `-` denied.

| Capability | Superadmin | Operations | Support | Auditor |
|---|---:|---:|---:|---:|
| Create tenant | A | - | - | - |
| Read tenants | A | A | A | R |
| Edit tenant | A | C | - | R |
| Change tenant status | A | C | - | R |
| Archive tenant | C | - | - | R |
| Reset owner access | A | - | C | R |
| Replace tenant owner | C | - | - | R |
| Start support session | A | - | A | R |
| Mutate during support session | C | - | C | R |
| Manage plans | A | - | - | R |
| Change tenant plan | A | - | - | R |
| Read usage | A | A | A | R |
| Manage global settings | A | - | - | R |
| Change global secrets | C | - | - | R |
| Manage feature flags | A | C | - | R |
| Maintenance mode | A | A | - | R |
| Read health/failures | A | A | R | R |
| Retry failed jobs | A | A | - | R |
| Read global audit | A | R | R | R |

Conditions:

- Operations cannot archive tenants.
- Owner replacement, tenant archival, global-secret changes, and emergency support mutations require dual control.
- Support mutations require `support.mutate_in_session`, active support session, explicit reason, and audit.

## 6. Tenant Administration Matrix

| Capability | Owner | Admin | Order Mgr | Warehouse Mgr | Staff | Support | Finance | Auditor |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| Read tenant/store/warehouse | A | A | R | R | R | R | R | R |
| Update tenant/store | A | A | - | - | - | - | - | - |
| Update warehouse | A | A | - | A | - | - | - | - |
| Invite/deactivate users | A | A | - | - | - | - | - | R |
| Assign roles | A | A | - | - | - | - | - | R |
| Read usage | A | A | R | R | R | R | R | R |
| Read tenant audit | A | A | C | C | - | C | C | A |

Rules:

- At least one active Tenant Owner must remain.
- A user cannot grant permissions above their own assignment ceiling.
- Global roles cannot be assigned through tenant APIs.
- Role changes invalidate authorization caches and may revoke sessions.

## 7. Catalog Matrix

| Capability | Owner | Admin | Order Mgr | Warehouse Mgr | Staff | Support | Finance | Auditor |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| Read catalog | A | A | R | R | R | R | R | R |
| Create/update product | A | A | - | - | - | - | - | - |
| Publish/deactivate/archive | A | A | - | - | - | - | - | R |
| Manage variants | A | A | - | R | R | - | - | R |
| Manage categories/images | A | A | - | - | - | - | - | R |

Additional guards: tenant ownership, plan limit, publish validation, feature availability, and archival rather than destructive deletion for historically referenced variants.

## 8. Order, Payment, Return, and Refund Matrix

| Capability | Owner | Admin | Order Mgr | Warehouse Mgr | Staff | Support | Finance | Auditor |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| Read orders | A | A | A | R | R | A | R | R |
| Read customer contact | A | A | A | C | C | A | R | R |
| Add internal note | A | A | A | R | - | A | - | R |
| Confirm manual payment | A | A | A | - | - | - | C | R |
| Request cancellation | A | A | A | C | C | A | - | R |
| Approve/reject cancellation | A | A | A | - | - | - | - | R |
| Approve/reject return | A | A | A | C | - | A | - | R |
| Create refund | A | A | A | - | - | - | C | R |
| Retry refund | A | A | A | - | - | - | C | R |
| Read timeline | A | A | A | A | A | A | R | R |

Rules:

- Finance Viewer remains read-only unless explicitly granted elevated financial permission.
- Support cannot confirm payments or execute refunds.
- Large refunds require configurable dual approval.
- State, amount, ownership, idempotency, and version guards always apply.

## 9. Inventory and WMS Matrix

| Capability | Owner | Admin | Order Mgr | Warehouse Mgr | Staff | Support | Finance | Auditor |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| Read inventory/movements | A | A | R | A | A | R | R | R |
| Adjust inventory | A | A | - | A | C | - | - | R |
| Create receiving | A | A | - | A | A | - | - | R |
| Complete/cancel receiving | A | A | - | A | C | - | - | R |
| Start/pause/resume picking | A | A | - | A | A | - | - | R |
| Report exception | A | A | C | A | A | A | - | R |
| Complete picking/packing | A | A | - | A | A | - | - | R |
| Resolve exception | A | A | C | A | C | - | - | R |
| Print list/label | A | A | A | A | A | A | - | R |

Warehouse Staff conditional actions require task assignment, configured quantity threshold, allowed movement type, or manager approval.

## 10. Shipment Matrix

| Capability | Owner | Admin | Order Mgr | Warehouse Mgr | Staff | Support | Finance | Auditor |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| Create shipment | A | A | A | A | C | - | - | R |
| Read tracking/label | A | A | A | A | A | A | R | R |
| Cancel shipment | A | A | A | C | - | - | - | R |
| Reconcile unknown state | A | A | A | C | - | C | - | R |

Shipment creation requires payment, picking, packing, package, tenant, and state guards. Support reconciliation defaults to recommendation-only unless elevated permission is assigned.

## 11. Reporting, Notifications, and Audit

| Capability | Owner | Admin | Order Mgr | Warehouse Mgr | Staff | Support | Finance | Auditor |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| Read dashboard | A | A | A | A | C | A | A | R |
| Create sales report | A | A | A | R | - | R | A | R |
| Create inventory report | A | A | R | A | C | R | R | R |
| Download report | A | A | C | C | - | C | A | R |
| Read own notifications | own | own | own | own | own | own | own | own |
| Read audit logs | A | A | C | C | - | C | C | A |

Report downloads require original request authorization or equivalent permission re-evaluation, expiring links, and export logging.

## 12. Customer Rules

Customers may only:

- browse the resolved tenant catalog
- manage their own cart and addresses
- claim a cart with a valid claim token
- checkout their own authenticated cart
- read their own orders, payments, shipments, returns, refunds, and notifications
- request cancellation or return only when state and policy permit
- receive SSE only for their own resources

## 13. Provider and Worker Rules

Biteship may only submit verified webhook requests. It has no general API authorization.

System workers use dedicated service identities with least privilege, carry tenant context, consume approved event families, and never use unrestricted database-superuser credentials.

## 14. Support Mode

Support mode requires:

- `support.session.start`
- explicit tenant and reason
- short expiry
- visible banner
- real actor and effective tenant recording
- separate permission for mutations
- no secret/password exposure
- no state-machine bypass
- dual control for emergency high-impact mutations

## 15. Separation of Duties

Configurable dual control is mandatory for:

- large refunds above configured threshold
- tenant archival
- primary owner replacement
- global secret changes
- emergency support-mode mutations

The initiator and approver must be different identities. Approval, rejection, expiry, and override are audited.

## 16. Sensitive Data Rules

- Password hashes and secrets are never returned.
- Customer contact data is need-to-know.
- Full shipping address is limited to order and fulfillment roles.
- Raw provider payloads are operations/security only.
- Internal notes are never customer-visible.
- Payment instruments are neither stored nor exposed beyond provider-safe metadata.

## 17. Deny and Audit Rules

Deny when tenant, ownership, membership, permission, plan, feature, state, version, support session, or dual-control requirements fail.

Mandatory audit includes tenant lifecycle, plan changes, support sessions, user/role changes, payment confirmation, cancellation/return/refund decisions, inventory adjustments, receiving completion, shipment creation/cancellation/reconciliation, feature flags, maintenance mode, privileged exports, and operational retries.

## 18. Acceptance Criteria

1. Every OpenAPI operation has a machine-checkable permission or audience mapping.
2. Conditional grants are denied by default.
3. State, version, tenant, and ownership guards apply consistently.
4. Customer access is self-only.
5. Dual control protects high-impact actions.
6. Support mode cannot silently elevate privileges.
7. Sensitive data follows least privilege.

## Change Summary

Final version closes SEC-M01 through SEC-M03 with deny-by-default conditional grants, a CI-enforced operation-permission registry, and dual-control requirements for high-impact actions.
