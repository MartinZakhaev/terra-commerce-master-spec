# ADR-0014 — Use Resend for Transactional Email

**Status:** Final  
**Date:** 2026-07-01  
**Decision Owners:** Product Owner, Platform Lead, Security Owner  
**Related:** FSD v1.0.1, Worker/Event Pipeline TDD v1.0.0, Environment Specification v1.0.0

## Context

Terra Commerce requires transactional invitation, password-recovery, order, payment, fulfillment, delivery, and security email without operating its own mail infrastructure.

## Decision

Use a platform-owned Resend account with separate API credentials for development, staging, and production.

MVP rules:

- Send from a verified platform-controlled subdomain.
- Tenant branding controls display name, template content, and approved reply-to address.
- Per-tenant custom sending domains are deferred.
- Email delivery is asynchronous through the worker.
- Local notification and delivery-attempt records are authoritative for deduplication and audit.
- Provider idempotency is used where supported, but never replaces durable local idempotency.
- API keys are least privilege, masked, versioned, and rotatable.
- Email failure never rolls back a committed business transaction.

## Alternatives Considered

- Self-hosted SMTP: rejected due to deliverability and operational overhead.
- Tenant-owned email accounts: rejected for MVP because onboarding and support complexity would be high.
- Other transactional providers: viable, but Resend is selected for a simple API and developer experience.

## Consequences

Positive: quick integration, provider-managed delivery infrastructure, and consistent tenant-branded templates.

Negative: shared platform sending reputation and dependency on one external provider.

## Review Trigger

Review if tenants require custom sending domains, dedicated IPs, regional routing, or provider redundancy.