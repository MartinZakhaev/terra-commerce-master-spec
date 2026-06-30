# Requirements Traceability Matrix — Terra Commerce

**Document ID:** QA-TC-002  
**Version:** 1.0.0  
**Status:** Final  
**Owner:** QA Lead and Product Owner  
**Last Updated:** 2026-06-30  
**Upstream:** PRD v1.0.0 Final and all finalized downstream specifications  
**Audit:** `reviews/Quality_Test_Strategy_Suite_Audit_v1.0.0.md`

## 1. Purpose

This matrix defines the authoritative traceability model connecting product requirements to functional behavior, architecture, state, data, contracts, authorization, security controls, technical designs, tests, evidence, and release gates.

## 2. Traceability Chain

```text
PRD Requirement
 -> FSD Requirement
 -> ADR / Architecture
 -> State Machine
 -> Data Design
 -> API / Event / SSE / Webhook Contract
 -> Authorization / Threat Control
 -> TDD Component
 -> Test Case / Evidence
 -> Release Gate
```

Every mandatory PRD acceptance criterion requires a complete chain or an approved, time-bounded exception.

## 3. Authoritative Reference Rule

Use existing registered identifiers whenever available:

- document IDs and versions
- FSD requirement IDs
- ADR numbers
- OpenAPI `operationId`
- event type and schema version
- permission key
- threat ID
- table/constraint name
- test case ID

For upstream sections without granular IDs, use `Document ID + version + section heading`. Do not invent identifiers that appear authoritative until they are added to the relevant registry/document.

Planned implementation identifiers such as detailed state-transition IDs or constraint IDs must be marked `planned` until formally registered.

## 4. Master Matrix Fields

| Field | Meaning |
|---|---|
| Requirement reference | Existing authoritative ID or document/section |
| Summary and priority | Required behavior and priority |
| Source version | Authoritative document version |
| Functional reference | FSD behavior |
| Architecture/ADR | Boundary or approved decision |
| State reference | Lifecycle section/invariant |
| Data reference | Table, field, constraint, index |
| Contract reference | operationId/event/SSE/webhook |
| Authorization reference | permission/audience/ownership |
| Threat/control reference | security/privacy control |
| TDD component | implementing module/service |
| Test cases | verification identities |
| Evidence | CI/report/artifact URI or immutable reference |
| Status | not covered, designed, implemented, verified, accepted |
| Owner | accountable role/team |

## 5. PRD Acceptance Baseline

| PRD Acceptance Criterion | Principal Downstream Coverage | Required Verification |
|---|---|---|
| Tenant create/activate/suspend/inspect | FSD tenant functions; tenant state; platform APIs; authorization | functional, state, audit, session/SSE termination |
| Tenant users and permissions | identity data; tenant APIs; authorization matrix | role ceiling, owner protection, revocation |
| Store and warehouse setup | tenant configuration and warehouse data | validation, tenant isolation, inactive guards |
| Product create/publish | catalog module, APIs, events | publish rules, plan limits, historical integrity |
| Browse/cart/rates/order | storefront APIs and checkout TDD | E2E, idempotency, provider failure, inventory |
| Safe inventory reservation | inventory state/data/module | race, exact stock, rollback, tenant constraints |
| Tenant order processing | OMS state/API/permissions | allowed/forbidden states, timeline, audit |
| Picking and packing | WMS state/data/API/TDD | assignment, exception, compensation |
| Biteship shipment/tracking | shipment state/adapter/webhook | timeout, duplicate, ordering, reconciliation |
| Idempotent webhooks | webhook event/idempotency/outbox | replay, hash conflict, transaction atomicity |
| Tenant-scoped SSE | SSE/gateway/worker contracts | audience, replay, revocation, load |
| Suspended tenant blocked | tenant/gateway/SSE rules | active-session and mutation denial |
| Critical actions audited | audit data/authorization/threat controls | completeness, sanitization, immutability |
| Cross-tenant tests | architecture/data/security | API, repository, worker, event, file, report |
| Target-host load test | NFR/performance/infrastructure | latency, resources, stability, recovery |
| Backup and restore | architecture/infrastructure | isolated restore and domain validation |
| Documents in master-spec | governance/release design | repository and version checks |
| Apps as submodules | ADR-0006/release TDD | exact pin and manifest checks |

## 6. Coverage Rules

### Functional Domains

Every platform, tenant, catalog, customer, cart, checkout, OMS, inventory, WMS, delivery, event, notification, reporting, audit, and operations requirement has a row.

### State Machines

Every allowed transition has a positive case. Every high-risk forbidden transition has a negative case. Every mutable aggregate has stale-version and concurrency cases.

### Contracts

Every OpenAPI operationId has success, authentication/audience, permission/ownership, validation, and declared error tests. Stateful commands add version, state, and idempotency tests.

Every event type has producer/schema/consumer/duplicate/version/sensitivity coverage. Every staff/customer SSE event family has audience and replay tests.

### Security

Every Critical and High threat maps to implemented controls, tests, monitoring/release gate, and owner. Residual risk acceptance records include owner, justification, expiry, and review trigger.

## 7. Status Workflow

- `not_covered`
- `designed`
- `implemented`
- `verified`
- `accepted`

A requirement cannot become accepted without implementation and current verification evidence.

## 8. Change Impact

When an upstream requirement changes:

1. Mark rows impacted.
2. Update downstream references.
3. Add/update/deprecate test cases.
4. Execute affected regression.
5. Attach evidence for the new version.
6. Preserve the release snapshot of the previous matrix.

## 9. Machine-Readable Registry

A YAML/CSV/JSON registry should be maintained during implementation and validated in CI for:

- duplicate or unknown references.
- use of unregistered authoritative-looking IDs.
- mandatory requirements without tests.
- protected operationIds without authorization/security tests.
- Critical/High threats without evidence.
- accepted risks without owner or expiry.

## 10. Acceptance Criteria

1. All 18 PRD acceptance criteria are represented.
2. Existing identifiers are used accurately; planned identifiers are clearly marked.
3. Mandatory domains, operations, transitions, and threats are traceable.
4. Evidence identifies exact release artifacts.
5. Changes preserve historical traceability.

## Change Summary

Final version closes QA-M01 by distinguishing registered authoritative references from planned implementation identifiers and prohibiting invented authoritative IDs.
