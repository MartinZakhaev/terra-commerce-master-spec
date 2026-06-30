# Security Threat Model — Terra Commerce

**Document ID:** SEC-TC-002  
**Version:** 1.0.0  
**Status:** Draft  
**Owner:** Security Owner and Technical Lead  
**Last Updated:** 2026-06-30  
**Upstream:** PRD v1.0.0 Final, FSD v1.0.1 Draft, System Architecture v1.0.0 Final, State Machine v1.0.0 Final, Database Design v1.0.0 Final, Contract Suite v1.0.0 Final, Authorization Matrix v1.0.0 Draft

## 1. Purpose

This threat model identifies trust boundaries, assets, attackers, abuse cases, mitigations, verification requirements, and accepted residual risks for Terra Commerce MVP.

## 2. Method

The model uses STRIDE categories:

- Spoofing
- Tampering
- Repudiation
- Information disclosure
- Denial of service
- Elevation of privilege

Risk rating combines likelihood and business impact:

- Critical
- High
- Medium
- Low

## 3. Security Objectives

1. Prevent cross-tenant data access.
2. Prevent unauthorized state transitions and financial actions.
3. Protect customer personal data, credentials, provider secrets, and audit history.
4. Ensure webhook, event, and background processing integrity.
5. Preserve availability within the constrained MVP infrastructure.
6. Detect and investigate sensitive actions and security failures.
7. Recover business truth from PostgreSQL and external backups.

## 4. Assets

### Critical assets

- Tenant isolation context.
- User and customer authentication credentials.
- Role and permission assignments.
- Orders, payment state, refunds, and inventory.
- Provider credentials and webhook verification material.
- PostgreSQL data and backups.
- Audit logs and state-transition history.
- Outbox and idempotency records.

### Sensitive assets

- Customer names, email, phone, and addresses.
- Shipping labels and tracking data.
- Internal notes and support-session records.
- Generated reports.
- Operational logs and failure details.

## 5. Trust Boundaries

1. Browser or external client to reverse proxy.
2. Reverse proxy to Go gateway.
3. Gateway to Medusa/platform modules.
4. Application services to PostgreSQL and Redis.
5. Worker to Redis Streams and providers.
6. Biteship to public webhook endpoint.
7. Application to object storage and email provider.
8. Global support mode into tenant context.
9. CI/CD and operators into production infrastructure.

## 6. Threat Actors

- Unauthenticated internet attacker.
- Malicious customer.
- Compromised tenant user.
- Malicious tenant administrator.
- Compromised global support or operations account.
- Supply-chain attacker.
- Compromised provider credential or forged webhook sender.
- Accidental developer/operator error.
- Automated abuse client or bot.

## 7. Threat Register

### TM-001 Cross-Tenant Object Access

**Category:** Information disclosure / Elevation of privilege  
**Risk:** Critical

Attack: actor changes resource IDs or tenant hints to read or mutate another tenant's data.

Controls:

- Server-resolved tenant context.
- Composite tenant-safe foreign keys.
- Tenant-aware repositories.
- Resource ownership validation downstream.
- Tenant-prefixed cache and object keys.
- Cross-tenant automated tests.
- No direct frontend access to Medusa or databases.

Verification:

- IDOR tests across all tenant-owned endpoints.
- Negative repository tests.
- SSE cross-tenant replay tests.
- Background job tenant-context tests.

### TM-002 Customer Access to Another Customer's Resources

**Category:** Information disclosure  
**Risk:** Critical

Controls:

- Customer ownership checks for orders, addresses, shipments, returns, refunds, notifications, and SSE.
- Non-enumerable IDs are helpful but not a substitute for ownership checks.
- Customer-specific replay scope.

### TM-003 Privilege Escalation Through Role Assignment

**Category:** Elevation of privilege  
**Risk:** High

Controls:

- Seeded roles for MVP.
- Assignment permission ceiling.
- Global roles cannot be tenant assigned.
- At least one owner remains.
- Role changes invalidate authorization cache and may revoke sessions.
- Role changes are audited.

### TM-004 Unauthorized Financial Transition

**Category:** Tampering / Elevation of privilege  
**Risk:** Critical

Targets: payment confirmation, cancellation approval, refund creation/retry.

Controls:

- Explicit permissions.
- State-machine guards.
- Aggregate version checks.
- Idempotency keys.
- Amount constraints.
- Audit logs and transition records.
- Support role denied by default.

### TM-005 Inventory Race or Oversell

**Category:** Tampering  
**Risk:** High

Controls:

- Row locks or atomic conditional updates.
- Versioned inventory aggregates.
- Reservation state machine.
- Non-negative database constraints.
- Idempotent reserve/release/consume.

### TM-006 Checkout Replay and Duplicate Orders

**Category:** Tampering / Denial of service  
**Risk:** High

Controls:

- Required idempotency key.
- Request hash binding.
- Atomic reservation and order creation.
- Partial unique indexes for tenant/global idempotency records.

### TM-007 Forged or Replayed Biteship Webhook

**Category:** Spoofing / Tampering  
**Risk:** Critical

Controls:

- Official provider authenticity verification.
- Raw-body verification before parsing trust.
- Provider event deduplication and payload hash.
- Tenant resolution from stored provider references.
- Monotonic shipment rules.
- Atomic tracking, state, transition, and outbox writes.
- Quarantine on conflict.

### TM-008 Domain Event Tampering or Duplicate Consumption

**Category:** Tampering / Repudiation  
**Risk:** High

Controls:

- Durable outbox in business transaction.
- Event IDs and aggregate versions.
- Consumer idempotency.
- Version-gap handling.
- Restricted Redis access.
- Dead-letter and reconciliation tooling.

### TM-009 SSE Data Leakage

**Category:** Information disclosure  
**Risk:** Critical

Controls:

- Authenticated connection.
- Tenant, customer, and permission filtering.
- Audience-scoped replay IDs.
- Periodic permission revalidation.
- Raw domain events never exposed.
- Bounded buffers and replay.
- Stream termination on suspension or revocation.

### TM-010 Session Theft or Fixation

**Category:** Spoofing  
**Risk:** High

Controls required from authentication ADR:

- Secure, HttpOnly, SameSite cookies if session based.
- Token rotation and revocation if token based.
- TLS only.
- Login session regeneration.
- Session invalidation after password reset, suspension, role-sensitive changes, and deactivation.
- MFA required for global privileged roles before production readiness.

### TM-011 Credential Stuffing and Authentication Abuse

**Category:** Spoofing / Denial of service  
**Risk:** High

Controls:

- Rate limiting by account and network signals.
- Progressive delay or lock policy.
- Generic authentication errors.
- Password hashing with approved adaptive algorithm.
- Breached-password checks if feasible.
- Security alerts for privileged accounts.

### TM-012 CSRF on Browser Mutations

**Category:** Tampering  
**Risk:** High when cookie sessions are used

Controls:

- SameSite cookies.
- CSRF token or origin verification.
- No state-changing GET requests.
- Approved-origin CORS.

### TM-013 XSS and Stored Content Injection

**Category:** Tampering / Information disclosure  
**Risk:** High

Targets: product descriptions, internal notes, branding, notifications.

Controls:

- Contextual output encoding.
- HTML sanitization for rich content.
- Content Security Policy.
- Safe URL validation.
- No raw HTML rendering from untrusted fields by default.

### TM-014 SQL/NoSQL/Command Injection

**Category:** Tampering / Elevation of privilege  
**Risk:** Critical

Controls:

- Parameterized queries and ORM safety.
- Input allowlists for sorting and filtering.
- No shell command construction from user input.
- Least-privilege database roles.
- Security tests for dynamic report/query features.

### TM-015 SSRF Through Provider, Media, or Webhook URLs

**Category:** Information disclosure / Elevation of privilege  
**Risk:** High

Controls:

- Do not fetch arbitrary user-provided URLs.
- Provider host allowlists.
- Block private, loopback, link-local, and metadata IP ranges.
- Validate redirects and DNS rebinding.
- Use controlled object-storage upload workflows.

### TM-016 Malicious File Upload

**Category:** Tampering / Denial of service  
**Risk:** High

Controls:

- File type and size allowlists.
- Content inspection where available.
- Randomized object keys.
- Private storage by default.
- Do not execute uploaded content.
- Image re-encoding where appropriate.
- Signed access for private reports and labels.

### TM-017 Secret Exposure

**Category:** Information disclosure  
**Risk:** Critical

Controls:

- Secrets outside source control.
- Separate environment/service credentials.
- Masked settings API.
- Log redaction.
- Rotation process.
- CI secret scanning.
- No secrets in SSE, events, reports, or error responses.

### TM-018 Audit Log Tampering

**Category:** Repudiation / Tampering  
**Risk:** High

Controls:

- Insert/select-only application permissions.
- No normal update/delete repository methods.
- Restricted maintenance role.
- Backup and retention.
- Correlation IDs and transition IDs.
- Alert on privileged maintenance access.

### TM-019 Support Mode Abuse

**Category:** Elevation of privilege / Repudiation  
**Risk:** Critical

Controls:

- Dedicated permission.
- Explicit tenant and reason.
- Short expiry.
- Visible banner.
- Real actor plus effective tenant.
- No password/secret exposure.
- Mutations require separate elevated permission.
- Session recording through audit events.
- MFA for global support roles.

### TM-020 Denial of Service Against Small VPS

**Category:** Denial of service  
**Risk:** High

Targets: login, search, reports, shipping rates, checkout, SSE, webhooks, file upload.

Controls:

- Route and identity rate limits.
- Request/body limits.
- Pagination and query limits.
- Async heavy reports.
- Worker concurrency caps.
- SSE connection and buffer limits.
- Database statement timeouts.
- Backpressure and circuit breakers.
- Resource alerts.

### TM-021 Redis Failure or Poisoning

**Category:** Denial of service / Tampering  
**Risk:** High

Controls:

- Redis private network access.
- Authentication and least privilege where supported.
- Memory limits and eviction policy.
- PostgreSQL remains authoritative.
- Consumer validation of event envelopes.
- Runbook for degraded cache, rate limiting, streams, and SSE.

### TM-022 Database Compromise or Backup Exposure

**Category:** Information disclosure / Tampering  
**Risk:** Critical

Controls:

- Private network only.
- Least-privilege service accounts.
- Encrypted transport where crossing hosts.
- Encrypted off-host backups.
- Restricted restore credentials.
- Restore testing.
- Backup access audit.

### TM-023 Supply-Chain Compromise

**Category:** Tampering / Elevation of privilege  
**Risk:** High

Controls:

- Lockfiles.
- Dependency and container scanning.
- Trusted registries.
- Minimal images.
- Signed or verified build artifacts where feasible.
- Protected branches and reviewed workflows.
- Submodule commit pinning.

### TM-024 CI/CD or Deployment Credential Abuse

**Category:** Elevation of privilege  
**Risk:** Critical

Controls:

- Short-lived deployment credentials where possible.
- Environment protection and approvals.
- Least privilege.
- Secret masking.
- No untrusted PR access to production secrets.
- Deployment audit trail and rollback.

## 8. Security Controls by Layer

### Reverse proxy

- TLS.
- Request limits.
- Security headers.
- Approved routing only.
- SSE buffering disabled only on stream route.

### Gateway

- Authentication, tenant resolution, route permissions, rate limiting, validation, correlation IDs, normalized errors.

### Domain modules

- Resource ownership, state guards, version checks, plan/feature checks, transaction integrity.

### Database

- Composite tenant-safe foreign keys.
- Constraints.
- Least-privilege roles.
- Append-only permissions.
- Encrypted backups.

### Redis and workers

- Private access.
- Idempotent consumers.
- Bounded retries.
- Dead letters.
- Tenant-context validation.

### Frontends

- Output encoding.
- CSP.
- No secrets.
- Secure session handling.
- UI does not substitute for backend authorization.

## 9. Security Logging and Alerting

Alert conditions:

- repeated authentication failures
- cross-tenant authorization denials
- privileged role or owner changes
- support-session start and mutation
- webhook verification failures or payload conflicts
- refund/payment anomalies
- repeated state conflicts
- secret scanning findings
- audit-table maintenance access
- backup failures
- resource exhaustion

Logs must redact credentials, payment-sensitive data, full tokens, secret headers, and unnecessary personal data.

## 10. Security Verification Plan

Required before production:

- authorization tests for every protected operation
- cross-tenant and cross-customer IDOR tests
- state-transition negative tests
- concurrency and idempotency tests
- webhook forgery/replay tests
- SSE audience and replay tests
- file-upload abuse tests
- rate-limit and resource-exhaustion tests
- dependency/container scans
- backup restore test
- privileged support-mode review
- external penetration test or focused independent review before broad production use

## 11. Residual Risks

Accepted for MVP with monitoring:

- Shared-schema multitenancy increases impact of application scoping defects.
- Single VPS creates availability and blast-radius concentration.
- Authentication strategy is not final until its ADR is approved.
- Provider-specific webhook verification cannot be finalized until current official Biteship documentation is implemented.
- Full RLS is deferred.

## 12. Security Release Gates

Release is blocked when:

- any Critical threat lacks implemented mitigation
- cross-tenant tests fail
- privileged endpoint lacks authorization mapping
- sensitive fields appear in logs/events/SSE
- backup restore has never passed
- webhook authenticity verification is not implemented
- critical dependency vulnerabilities remain unaccepted
- production secrets exist in source control

## 13. Acceptance Criteria

1. Every critical asset and trust boundary is represented.
2. Critical tenant, financial, webhook, SSE, support, and infrastructure threats have controls and tests.
3. Authorization Matrix and threat controls agree.
4. Residual risks are explicit.
5. Security release gates are testable.

## Change Summary

Initial detailed threat model synchronized with the final architecture, contracts, data model, state machines, and authorization rules.
