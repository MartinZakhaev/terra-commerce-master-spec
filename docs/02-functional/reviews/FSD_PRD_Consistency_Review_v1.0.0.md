# PRD–FSD Consistency and Completeness Review

**Document ID:** REVIEW-TC-001  
**Version:** 1.0.0  
**Status:** Completed  
**Review Date:** 2026-06-30  
**Reviewed PRD:** `docs/01-product/prd/PRD_Terra_Commerce_v1.0.0_Final.md`  
**Reviewed FSD:** `docs/02-functional/fsd/FSD_Terra_Commerce_v1.0.0_Draft.md`

---

## 1. Executive Summary

The FSD is broadly consistent with the approved PRD and correctly carries forward the locked MVP architecture, tenancy model, major actors, OMS, WMS, Biteship, SSE, security, audit, reporting, and infrastructure constraints.

No direct contradiction was found in the locked technology stack or tenant model.

However, the FSD is not yet complete enough to move to In Review. The audit identified:

- 4 high-priority functional gaps.
- 10 medium-priority completeness gaps.
- 4 low-priority documentation and traceability gaps.
- 1 material ambiguity involving guest checkout.

**Audit result:** Conditionally aligned, revision required before FSD promotion.

---

## 2. Review Method

The review compared the PRD and FSD across:

1. Product scope and non-goals.
2. Actors and roles.
3. Tenant lifecycle and multitenancy.
4. Superadmin capabilities.
5. Tenant administration.
6. Catalog and storefront.
7. OMS and WMS.
8. Biteship integration.
9. SSE and notifications.
10. Security and audit.
11. Reporting and observability.
12. Non-functional requirements.
13. Acceptance criteria and traceability.

Severity definitions:

- **High:** Required PRD behavior is missing or ambiguous enough to block downstream design.
- **Medium:** Requirement is partially covered but lacks sufficient functional detail.
- **Low:** Documentation, traceability, or clarity issue that does not immediately block architecture.

---

## 3. Confirmed Consistencies

### 3.1 Locked MVP Architecture

The FSD correctly preserves:

- Go gateway.
- Medusa.js v2.
- PostgreSQL.
- Redis Streams.
- SSE.
- Biteship.
- Modular monolith.
- 2 vCPU and 4 GB RAM target.

### 3.2 Tenant Model

Both documents consistently define:

- One tenant equals one business.
- One tenant has one store.
- One tenant has one warehouse.
- Shared infrastructure with logical tenant isolation.

### 3.3 Core Business Domains

The FSD provides functional coverage for:

- Tenant provisioning and status changes.
- Tenant users and role assignment.
- Product creation and tenant-scoped listing.
- Customer registration, login, and addresses.
- Cart, shipping rates, and checkout.
- Order viewing, payment confirmation, cancellation, returns, and refunds.
- Inventory reservation and adjustment.
- Picking, packing, and warehouse exceptions.
- Biteship shipment creation and webhook handling.
- SSE authentication, reconnection, and audience filtering.
- Audit logging.
- Tenant and global reporting.
- Retry and dead-letter handling.

### 3.4 Security and Isolation

The FSD correctly converts PRD principles into server-side tenant-context, permission, resource-ownership, business-state, rate-limit, and sensitive-data requirements.

---

# 4. High-Priority Findings

## FINDING-H01 — Subscription Plan Management Is Missing

**PRD requirement:** Superadmin must create plans, configure plan limits, assign and change tenant plans, and view usage.

**FSD coverage:** Tenant creation accepts a plan and reporting mentions plan usage, but there is no functional requirement for plan CRUD, plan assignment changes, limit enforcement, or limit-exceeded behavior.

**Impact:** Tenant provisioning, authorization, usage enforcement, data model, and billing architecture cannot be finalized.

**Required remediation:** Add requirements for:

- Create, edit, activate, and archive plans.
- Assign and change tenant plan.
- Define product, order, user, storage, SSE, module, and report limits.
- Enforce hard and soft limits.
- Define upgrade, downgrade, and limit-exceeded behavior.
- Audit plan changes.

## FINDING-H02 — Global Platform Configuration and Monitoring Are Incomplete

**PRD requirement:** Superadmin manages global settings and feature flags and sees platform health, failed jobs, failed Biteship requests, SSE connections, CPU, memory, disk, PostgreSQL, Redis, and alerts.

**FSD coverage:** Global reporting and observability are described generally, but no actor flows or permissions exist for viewing operational health, changing global configuration, maintenance mode, or feature flags.

**Impact:** Superadmin dashboard scope and platform API boundaries remain undefined.

**Required remediation:** Add functional requirements for:

- Global settings management.
- Feature-flag management.
- Maintenance mode.
- Platform health dashboard.
- Failed-job and integration-failure inspection.
- Retry or acknowledgement actions.
- Global alert visibility.

## FINDING-H03 — Inventory Receiving and Incoming Stock Workflow Are Missing

**PRD requirement:** Warehouse staff handle inventory receiving, and WMS supports incoming stock and stock-receiving movements.

**FSD coverage:** Incoming stock is listed as a quantity and stock receiving is listed as a movement type in the PRD, but the FSD has no receiving workflow.

**Impact:** WMS scope, inventory transitions, data design, and warehouse UI cannot be completed.

**Required remediation:** Add a stock-receiving requirement covering:

- Create receiving record.
- Select SKU and quantity.
- Confirm received quantity.
- Record damaged or rejected quantity.
- Update incoming and physical stock.
- Create inventory movement.
- Audit actor and reason.

## FINDING-H04 — Guest Checkout Behavior Is Internally Ambiguous

**PRD requirement:** Guest checkout is optional and remains an unresolved decision.

**FSD coverage:** `FSD-FR-040` states customers or guests may create a cart, while guest checkout remains listed as an open decision.

**Impact:** Authentication, customer identity, cart ownership, checkout, order ownership, and API contracts may diverge.

**Required remediation:** Separate guest cart from guest checkout and state one of:

- Guest cart allowed, checkout requires login; or
- Guest checkout enabled; or
- Guest checkout disabled for MVP.

Until resolved by ADR or product decision, checkout requirements must not imply guest completion.

---

# 5. Medium-Priority Findings

## FINDING-M01 — Tenant Owner Access Reset Is Missing

The PRD explicitly requires superadmin tenant-owner access reset. Add reset, revoke, resend invitation, session invalidation, audit, and security-notification behavior.

## FINDING-M02 — Superadmin Tenant Editing Is Not Explicit

Tenant creation and status changes exist, but editing tenant legal name, display name, owner contact, plan, timezone, and operational metadata is not defined as a superadmin flow.

## FINDING-M03 — Product Lifecycle Is Only Partially Defined

Product creation exists, but explicit update, publish, deactivate, archive, variant management, image management, category assignment, and validation flows are not independently specified.

## FINDING-M04 — Storefront Post-Checkout Functions Need Explicit Requirements

The PRD requires payment-status display, order confirmation, order history, and shipment tracking. Tracking is present, but confirmation, payment-status display, and customer order-history behavior are not explicit functional requirements.

## FINDING-M05 — Tenant Dashboard Omits Out-of-Stock Metric

The FSD dashboard metrics include low stock but not the PRD-required out-of-stock product count or list.

## FINDING-M06 — Order Timeline Behavior Is Underspecified

The order detail says a timeline exists, but does not define required entries, ordering, actor visibility, customer-visible versus internal entries, or immutability.

## FINDING-M07 — Picking List and Shipping Label Printing Are Missing

The PRD explicitly includes printing picking lists and labels where supported. The FSD describes picking and stores label URLs but does not define print or download behavior.

## FINDING-M08 — Notification Coverage Is Partial

The PRD includes new order, cancellation, low stock, picking exception, and user invitation notifications. The FSD email list omits several of these and in-app notification rules do not enumerate event coverage or recipient roles.

## FINDING-M09 — Delivery Status Mapping Is Not Explicit

The PRD defines normalized delivery statuses. The FSD references normalized status but does not list the full canonical set or mapping expectations. This must be fixed before the shipment state machine and webhook contract.

## FINDING-M10 — Tenant Warehouse Configuration Is Too Shallow

The PRD requires warehouse name, address, contact, phone, operational status, Biteship origin area, and operating hours. The FSD only states that warehouse origin must be complete, without a dedicated warehouse-configuration flow.

---

# 6. Low-Priority Findings

## FINDING-L01 — PRD Acceptance Criteria Are Not Fully Mapped

The FSD includes only a small number of explicit acceptance criteria. Create a complete mapping for all 18 PRD MVP acceptance criteria.

## FINDING-L02 — Requirement Traceability Is Section-Level Only

The current traceability table maps document areas, not stable requirement IDs. Add PRD requirement IDs or a traceability matrix before final approval.

## FINDING-L03 — PRD Non-Goals Are Not Fully Repeated

The FSD excludes major non-goals but omits several, including international shipping, supplier management, enterprise BI, custom-domain provisioning, cross-tenant sharing, offline WMS, and separate OMS/WMS services. These are not contradictions, but restating them avoids scope creep.

## FINDING-L04 — Repository and Submodule Acceptance Is Not Represented

The PRD includes repository and submodule acceptance criteria. These are governance and release concerns rather than runtime functions, but the FSD should reference them in non-functional or release acceptance traceability.

---

# 7. Coverage Matrix

| PRD Domain | FSD Coverage | Result |
|---|---|---|
| Product scope and locked stack | Explicit | Pass |
| Tenant model and isolation | Explicit | Pass |
| Tenant provisioning | Detailed | Pass |
| Tenant status lifecycle | Partial pending state machine | Pass with follow-up |
| Superadmin plan management | Missing | Fail |
| Superadmin global configuration | Partial | Fail |
| Superadmin monitoring | Partial | Fail |
| Tenant settings | Partial | Needs revision |
| Tenant users and roles | Detailed | Pass |
| Catalog lifecycle | Partial | Needs revision |
| Storefront browsing | Detailed | Pass |
| Customer account | Partial | Needs revision |
| Cart and checkout | Detailed but guest ambiguity | Needs decision |
| OMS | Broad coverage | Pass with refinement |
| Inventory reservation and adjustment | Detailed | Pass |
| Inventory receiving | Missing | Fail |
| Picking and packing | Detailed | Pass |
| Biteship rates and shipment | Detailed | Pass |
| Biteship tracking statuses | Partial | Needs revision |
| SSE | Detailed | Pass |
| Notifications | Partial | Needs revision |
| Audit | Detailed | Pass |
| Reporting | Broad coverage | Pass |
| Non-functional requirements | Mostly aligned | Pass with downstream detail |
| PRD acceptance traceability | Partial | Needs revision |

---

# 8. Recommended FSD Revision Order

Apply changes in this order:

1. Resolve guest checkout policy or explicitly separate guest cart from checkout.
2. Add subscription plan management and enforcement.
3. Add global platform settings, feature flags, maintenance, and health monitoring.
4. Add warehouse configuration and inventory receiving.
5. Expand catalog lifecycle operations.
6. Add customer order history, payment-status display, and order confirmation.
7. Define order timeline requirements.
8. Add picking-list and shipping-label printing.
9. Complete notification event and recipient mapping.
10. Add canonical delivery statuses.
11. Complete PRD acceptance and requirement traceability.
12. Restate all MVP non-goals.

---

# 9. Promotion Gate

The FSD must remain **Draft** until all High findings are resolved.

It may move to **In Review** when:

- H01 through H04 are resolved.
- Medium findings are either resolved or explicitly assigned to downstream specifications without losing required product behavior.
- PRD acceptance criteria have traceable FSD coverage.
- Open decisions have owners and target ADRs.

It may move to **Approved/Final** only after architecture feasibility review, state-machine review, authorization review, and Product Owner approval.

---

# 10. Conclusion

The current FSD is a strong initial draft and is materially aligned with the PRD. It should not yet be used as a complete implementation contract because several required superadmin, WMS, storefront, and plan-management behaviors are absent or ambiguous.

The next action is to revise the FSD to address this report, producing `FSD_Terra_Commerce_v1.0.1_Draft.md` or updating the existing v1.0.0 Draft before it reaches In Review, according to the governance versioning policy.
