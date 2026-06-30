# Multitenancy, Authorization, and Security Test Plan — Terra Commerce

**Document ID:** QA-TC-004  
**Version:** 1.0.0  
**Status:** Final  
**Owner:** Security Owner and QA Lead  
**Last Updated:** 2026-06-30  
**Upstream:** Final Architecture, Database Design, Contract Suite, Authorization Matrix, Threat Model, and TDD Suite  
**Audit:** `reviews/Quality_Test_Strategy_Suite_Audit_v1.0.0.md`

## 1. Purpose

This plan verifies tenant isolation, customer ownership, authorization completeness, sensitive-data controls, webhook and SSE security, application/infrastructure threats, privacy, incident readiness, and all security release gates.

## 2. Execution Safety

Intrusive DAST, fuzzing, SSRF, malicious-upload, resource-exhaustion, and denial-of-service tests run only in approved non-production environments.

Any production security exercise requires separately approved scope, owner, maintenance/rollback plan, traffic limits, evidence handling, and abort conditions. Unscheduled destructive testing against production is prohibited.

## 3. Tenant Isolation

For every tenant-owned API, repository, cache, object, job, event, report, audit record, and SSE projection:

- Use overlapping identifiers in tenant A and B.
- Attempt path/query/body/header spoofing.
- Attempt direct and indirect child-resource access.
- Verify tenant-safe database constraints.
- Verify search/list/export cannot cross tenant.
- Verify support mode retains explicit tenant context.

Mandatory domains include administration, catalog, customers, carts, orders, payments, inventory, WMS, shipments, returns, refunds, notifications, reports, audit, transitions, and outbox.

## 4. Customer Ownership

Customers cannot access another customer's carts, addresses, orders, payments, shipments, returns, refunds, notifications, reports, or SSE replay. Cart claim tokens are single-use/rotated and tenant/customer scoped.

## 5. Authorization Registry

CI verifies every OpenAPI operationId has exactly one primary permission or explicit public/customer/provider rule, valid permission reference, state/version/ownership metadata, and no orphan or conflicting entry.

Runtime tests cover every default role, conditional deny-by-default grant, assignment ceiling, role revocation, last-owner protection, and support-session rule.

## 6. High-Impact Authorization

Verify:

- Finance Viewer cannot mutate financial state by default.
- Warehouse Staff remains within assignment/threshold policy.
- Global Operations cannot archive tenants.
- Support cannot mutate without active session and separate permission.
- Initiator cannot approve own dual-control action.
- Approval expiry and changed underlying state invalidate execution.

## 7. State, Version, and Idempotency Abuse

- Unauthorized valid transition.
- Forbidden source state.
- Stale version.
- Same idempotency key same/different payload.
- Concurrent payment confirmations, refunds, shipments, reservations, or approvals.
- Amount and quantity bounds after retries.

## 8. Authentication and Session Security

Once the authentication ADR is final, release-blocking tests cover issuance, fixation prevention, expiry, logout/reset/suspension/deactivation revocation, privileged MFA, cookie/CSRF or token rotation, credential stuffing, and generic authentication errors.

## 9. Webhook and SSE Security

Biteship:

- valid/invalid authenticity proof.
- raw-body mutation.
- replay and changed-payload conflict.
- tenant spoofing.
- malformed/oversized input.
- unknown/out-of-order/terminal-conflict state.
- no false success without processing or durable retry.

SSE:

- authentication and audience filtering.
- audience-scoped replay.
- permission revocation and suspension.
- slow consumer and connection limits.
- payload sensitivity.
- logout/expiry/shutdown termination.

## 10. Application Security

Test SQL/query/command/log injection, stored/reflected XSS, CSRF according to auth model, SSRF, path/object-key manipulation, malicious file uploads, unsafe redirects, and report/query abuse.

All dynamic tests use approved non-production targets unless separately authorized as described in Section 2.

## 11. Sensitive Data and Privacy

- Secrets and password hashes never returned/logged.
- Role-based need-to-know projection for customer fields.
- Events/SSE/reports/audit sanitize PII.
- Report download revalidates current permission and ownership.
- Signed links expire.
- Bulk export has separate permission and audit.
- Retention jobs preserve required history and remove eligible data.
- High-volume personal-data access is observable.

## 12. Database and Infrastructure

Verify private PostgreSQL/Redis exposure, least-privilege service roles, append-only update/delete denial, separate migration identity, encrypted backup controls, unprivileged containers, no container-socket exposure, host firewall, TLS, secret absence, and CI security scans.

## 13. Audit and Non-Repudiation

Mandatory audited actions verify actor, effective tenant, action, resource, sanitized before/after, correlation ID, timestamp, support/dual-control context, and immutable normal access.

## 14. Abuse and Capacity Security

In approved test environments, exercise authentication bursts, expensive search/report queries, checkout/rate/webhook/SSE bursts, file-size abuse, and database timeouts. Validate rate limits, backpressure, bounded buffers, alerts, and absence of state corruption.

## 15. Incident and Rotation Exercises

Exercise compromised support session, provider secret, database credential, customer session, cross-tenant alert, and malicious dependency scenarios. Verify revocation, rotation, evidence preservation, communication, validation, and post-incident tracking.

## 16. Exit Criteria

- Every Critical control has passing evidence.
- No Critical/High unaccepted defect.
- Cross-tenant/customer suite passes.
- Permission registry is complete.
- Webhook/SSE tests pass.
- Security owner approves.
- Intrusive test execution complied with environment-safety rules.

## Change Summary

Final version closes QA-M04 by explicitly separating intrusive security testing from production and defining the authorization required for any scoped production exercise.
