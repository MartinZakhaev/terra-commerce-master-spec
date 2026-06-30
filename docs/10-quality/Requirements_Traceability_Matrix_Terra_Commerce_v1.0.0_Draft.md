# Requirements Traceability Matrix — Terra Commerce

**Document ID:** QA-TC-002  
**Version:** 1.0.0  
**Status:** Draft  
**Owner:** QA Lead and Product Owner  
**Last Updated:** 2026-06-30  
**Upstream:** PRD v1.0.0 Final and all finalized downstream specifications

## 1. Purpose

This matrix defines the mandatory traceability model connecting product requirements to functional behavior, architecture, state, data, contracts, authorization, security controls, technical designs, and verification evidence.

## 2. Traceability Chain

```text
PRD Acceptance Criterion
 -> FSD Requirement
 -> Architecture / ADR
 -> State Machine
 -> Database Entity / Constraint
 -> API / Event / SSE / Webhook Contract
 -> Permission / Threat Control
 -> TDD Component
 -> Test Case / Evidence
 -> Release Gate
```

Every mandatory PRD acceptance criterion must have a complete chain or an approved documented exception.

## 3. Identifier Rules

Required identifiers:

- `PRD-AC-###` product acceptance criterion.
- `FSD-FR-###` functional requirement.
- `FSD-NFR-###` non-functional requirement.
- `ADR-####` architecture decision.
- `SM-<DOMAIN>-###` transition or invariant.
- `DATA-<TABLE/CONSTRAINT>` data requirement.
- OpenAPI `operationId`.
- Event type and schema version.
- Permission key.
- Threat ID `TM-###`.
- Test case ID `TC-<LEVEL>-<DOMAIN>-###`.

Existing documents without granular IDs must be mapped by stable document ID and section until explicit IDs are introduced.

## 4. Master Matrix Fields

| Field | Meaning |
|---|---|
| Requirement ID | Stable upstream identity |
| Requirement summary | Concise behavior or quality goal |
| Priority | Must/Should/Could or release priority |
| Source document/version | Authoritative source |
| FSD reference | Functional behavior |
| Architecture/ADR reference | Architectural implementation boundary |
| State reference | Required lifecycle behavior |
| Data reference | Entities, constraints, indexes, history |
| Contract reference | API/event/SSE/webhook operation |
| Authorization reference | Permission, audience, ownership rule |
| Threat/control reference | Security/privacy mitigation |
| TDD component | Implementing service/module |
| Test case IDs | Verification set |
| Evidence location | CI/report/artifact reference |
| Status | Not covered, designed, implemented, verified, accepted |
| Owner | Accountable person/team |

## 5. Baseline PRD Acceptance Mapping

| PRD Acceptance | Functional/Design Coverage | Verification Family |
|---|---|---|
| Create, activate, suspend, inspect tenant | FSD FR-001–005; tenant state machine; platform APIs; tenant permissions | Functional, authorization, state, audit |
| Manage tenant users and permissions | FSD FR-032–033; identity tables; authorization matrix | Functional, privilege escalation, session revocation |
| Configure store and warehouse | FSD FR-030–031; store/warehouse data; tenant APIs | Functional, validation, tenant isolation |
| Create and publish products | FSD FR-040–047; catalog module; product APIs/events | Functional, plan limits, publish validation |
| Browse, cart, rates, order | FSD FR-053, 060–062; checkout TDD; storefront API | E2E, idempotency, inventory, provider failure |
| Safe tenant-isolated reservation | inventory design/state; composite FK; repository rules | Concurrency, negative tenant, constraint |
| Tenant staff process orders | OMS requirements/state/API/permissions | Functional, state, authorization |
| Warehouse picking and packing | WMS state/data/API/TDD | E2E, task assignment, exception paths |
| Create and track Biteship shipment | shipment state, adapter, webhook contract | Integration, duplicate, timeout, ordering |
| Idempotent delivery webhooks | webhook/idempotency/outbox design | Replay, payload conflict, atomicity |
| Authenticated tenant-scoped SSE | SSE contract/gateway/worker design | Audience, replay, revocation, load |
| Suspended tenant blocked | tenant state, gateway checks, SSE close | Authorization, active-session termination |
| Critical actions audited | audit tables, authorization rules, threat model | Audit completeness and immutability |
| Cross-tenant tests pass | tenant architecture, data model, threat TM-001 | Dedicated isolation suite |
| Load test target server | NFR/TDD resource profile | Performance/load suite |
| Backup and restore tested | architecture/TDD backup design | Restore drill |
| Documents stored in master-spec | governance/release architecture | Repository validation |
| Apps linked as submodules | ADR-0006/release design | Submodule and manifest validation |

## 6. Functional Domain Coverage

The matrix must contain rows for:

- Platform and tenant lifecycle.
- Plans, usage, settings, flags, maintenance, support mode.
- Tenant users, roles, permissions, owner recovery.
- Catalog and product lifecycle.
- Customer identity, addresses, cart, checkout.
- Orders, payments, cancellation, timeline, return, refund.
- Inventory, receiving, picking, packing, exceptions.
- Shipments, tracking, labels, reconciliation.
- Events, notifications, reports, SSE.
- Audit, jobs, dead letters, operational retries.

## 7. State Coverage Rule

Every allowed transition requires at least one positive test. Every forbidden high-risk transition requires at least one negative test. Every stateful aggregate requires concurrency and stale-version verification.

## 8. Contract Coverage Rule

Every OpenAPI operationId requires:

- valid success test.
- authentication/audience test.
- permission/ownership test.
- input validation test.
- declared error tests.
- state/idempotency/version tests where applicable.

Every event type requires schema, producer, consumer, duplicate, version, and sensitivity tests. Every customer/staff SSE event requires audience tests.

## 9. Security Traceability Rule

Every Critical and High threat requires:

- implemented control reference.
- verification case.
- release gate or monitoring control.
- accountable owner.

Residual risks must link to an approved acceptance record and expiry/review trigger.

## 10. Status Management

- Not covered: no downstream design/test.
- Designed: mapped to approved design.
- Implemented: code/migration exists.
- Verified: required automated/manual evidence passes.
- Accepted: release authority signs off.

No requirement can move directly from Designed to Accepted without implementation and verification evidence.

## 11. Change Control

When an upstream requirement changes:

1. Mark affected rows impacted.
2. Update downstream references.
3. Identify obsolete and new test cases.
4. Re-run affected suites.
5. Record evidence against the new version.
6. Preserve the prior matrix snapshot with the release.

## 12. Machine-Readable Requirement

The implementation phase should maintain a machine-readable registry, such as YAML or CSV, containing the matrix fields and validated in CI for:

- unknown references.
- duplicate IDs.
- mandatory requirements without tests.
- protected operationIds without authorization tests.
- Critical/High threats without evidence.

## 13. Acceptance Criteria

1. All 18 PRD acceptance criteria are represented.
2. Every mandatory functional domain has design and test mappings.
3. Every protected operation and state transition is traceable.
4. Critical and High threats have evidence links.
5. Matrix status is tied to exact release commits and artifacts.

## Change Summary

Initial requirements traceability model and baseline mapping for Terra Commerce.
