# Multitenancy, Authorization, and Security Test Plan — Terra Commerce

**Document ID:** QA-TC-004  
**Version:** 1.0.0  
**Status:** Draft  
**Owner:** Security Owner and QA Lead  
**Last Updated:** 2026-06-30  
**Upstream:** Final Architecture, Database Design, Contract Suite, Authorization Matrix, Security Threat Model, and TDD Suite

## 1. Purpose

This plan verifies tenant isolation, customer ownership, authorization completeness, sensitive-data controls, webhook and SSE security, common application threats, infrastructure hardening, privacy, incident readiness, and the security release gates defined by the final threat model.

## 2. Test Case Convention

IDs use:

- `TC-SEC-TENANT-###`
- `TC-SEC-AUTHZ-###`
- `TC-SEC-APP-###`
- `TC-SEC-INFRA-###`
- `TC-SEC-PRIVACY-###`

Every Critical and High threat must map to one or more test cases and evidence.

## 3. Tenant Isolation Coverage

For every tenant-owned endpoint, repository, cache, file, job, event, report, and SSE projection:

- Create identical natural identifiers in tenant A and B.
- Attempt access using A identity with B resource ID.
- Attempt body/path/query/header tenant spoofing.
- Attempt indirect child access without parent ownership.
- Verify composite tenant-safe foreign-key rejection.
- Verify list/search filters cannot return another tenant.
- Verify support mode retains effective tenant and cannot silently cross context.

Mandatory domains:

- tenants/stores/warehouses
- users/memberships/roles
- products/variants/media
- customers/carts/addresses
- orders/payments/notes/timeline
- inventory/reservations/movements
- receiving/picking/packing/exceptions
- shipments/tracking/labels
- returns/refunds
- notifications/reports/audit/transitions/outbox

## 4. Customer Ownership Tests

- Customer cannot read or mutate another customer's cart, address, order, payment, shipment, return, refund, notification, or replay stream.
- Cart claim token cannot be reused, guessed, or used across tenants.
- Customer SSE replay ID cannot retrieve another customer's events.
- Customer-facing status never exposes internal notes, audit fields, or raw provider data.

## 5. Operation-to-Permission Registry Tests

CI validates:

- Every OpenAPI operationId has one primary permission or explicit public/customer/provider policy.
- No duplicate/conflicting mapping.
- Every permission exists in the seeded registry.
- Removed operations do not leave active orphan mappings.
- Every state-changing operation includes state/version/ownership metadata.
- Conditional grants are denied without explicit assignment/policy.

Runtime tests exercise allow/deny behavior for every role and operation family.

## 6. Role and Privilege Escalation

- Tenant admin cannot assign global roles.
- Actor cannot grant above own permission ceiling.
- Last active owner cannot be removed.
- Role revocation invalidates authorization cache and protected sessions.
- Finance Viewer cannot confirm payment or refund by default.
- Warehouse Staff threshold/assignment restrictions.
- Support cannot mutate without separate permission and active session.
- Global Operations cannot archive tenants.
- Dual-control initiator cannot approve own action.

## 7. State, Version, and Idempotency Security

- Unauthorized actor cannot exploit valid state transition.
- Stale version returns state conflict without side effect.
- Duplicate command with same idempotency key returns prior result.
- Same key with different payload is rejected.
- Race two approvals, refunds, shipments, or reservations and verify one valid outcome.
- Ensure retries cannot exceed captured/refundable or available quantities.

## 8. Authentication and Session Tests

After authentication ADR is final, verify:

- secure session/token issuance and expiry
- session fixation prevention
- logout and password-reset revocation
- tenant suspension and membership deactivation revocation
- privileged MFA enforcement
- cookie flags/CSRF or token rotation according to selected model
- brute-force and credential-stuffing limits
- generic authentication errors

These tests are release-blocking before production even if authentication implementation is deferred during early scaffolding.

## 9. Biteship Webhook Security

- Valid official authenticity proof.
- Invalid/missing/replayed signature or token.
- Raw-body mutation.
- Duplicate event same payload.
- Duplicate event changed payload.
- Spoofed body tenant ID.
- Unknown provider order ID.
- Oversized/malformed payload.
- Out-of-order and conflicting terminal states.
- Verify success is not returned without processing or durable retry guarantee.
- Verify secrets are absent from logs.

## 10. SSE Security

- Authentication required.
- Tenant/customer/permission filtering.
- Audience-scoped `Last-Event-ID`.
- Replay after role revocation.
- Replay after tenant suspension.
- Slow consumer and buffer exhaustion.
- Event payload sensitivity checks.
- CORS and credential handling.
- Stream termination on logout, expiry, revocation, suspension, shutdown.

## 11. Application Security Tests

### Injection

- SQL injection in filters, search, sorting, IDs, reports.
- Command/template injection in export and provider integration paths.
- Header and log injection.

### XSS

- Product descriptions, names, internal notes, branding, notifications, tracking descriptions.
- Validate sanitization, output encoding, and CSP behavior.

### CSRF

- All browser mutations when cookie sessions are used.
- Origin and token validation.
- No state-changing GET.

### SSRF

- Product/media URLs, provider callbacks, object references, redirects, private IPs, metadata IPs, DNS rebinding.

### File Upload

- MIME/extension mismatch.
- Oversized and decompression-bomb files.
- Executable content.
- Path traversal/object-key injection.
- Public/private access policy.

## 12. Sensitive Data and Privacy Tests

- Secrets/password hashes never returned or logged.
- Customer fields projected by role need-to-know.
- Reports/downloads revalidate permission and ownership.
- Signed links expire.
- Bulk exports require separate permission and are audited.
- Events/SSE omit excessive PII.
- Audit before/after data is sanitized.
- Retention jobs remove or archive eligible data and preserve required historical records.
- High-volume personal-data access triggers monitoring where configured.

## 13. Database and Infrastructure Security

- PostgreSQL and Redis ports are not public.
- Service accounts have expected least privileges.
- Append-only tables reject update/delete from application roles.
- Migration identity is separate.
- Backups are encrypted and access controlled.
- Containers do not run privileged or expose container socket.
- Secrets absent from source, images, logs, and lower environments.
- Host firewall and TLS configuration validated.
- Dependency/container/secret scans run in CI.

## 14. Audit and Non-Repudiation

For every mandatory audited action, verify:

- real actor and effective tenant
- action/resource
- before/after where applicable
- correlation ID
- timestamp
- support/dual-control context
- immutable normal access

Attempt audit deletion/update and unauthorized reading.

## 15. Abuse and Denial-of-Service Tests

- Authentication bursts.
- Catalog/search query abuse.
- Shipping-rate and checkout bursts.
- Webhook floods.
- SSE connection floods and tab multiplication.
- Report generation abuse.
- Upload size abuse.
- Expensive filters and database statement timeout.

Validate rate limits, bounded queues, connection caps, backpressure, and alerts without corrupting business state.

## 16. Incident and Rotation Exercises

Tabletop or controlled exercises:

- Compromised global support session.
- Leaked Biteship verification secret.
- Leaked application database credential.
- Customer session compromise.
- Cross-tenant access alert.
- Malicious dependency finding.

Verify revocation, rotation, evidence preservation, communication ownership, recovery validation, and post-incident action tracking.

## 17. Tooling and Execution

Use automated API/component tests, static analysis, dependency/container/secret scanners, dynamic security tests, infrastructure checks, and focused manual review. External penetration testing or independent security review is required before broad production use.

## 18. Release Exit Criteria

- All Critical threat controls tested and passing.
- No Critical/High unaccepted security defect.
- Cross-tenant/customer suite passes fully.
- Permission registry complete.
- Webhook and SSE security tests pass.
- Restore and incident/rotation readiness evidence current.
- Security owner approves release gate.

## Change Summary

Initial multitenancy, authorization, and security test plan synchronized with the final authorization and threat-model baseline.
