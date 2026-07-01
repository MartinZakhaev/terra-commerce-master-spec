# ADR-0009 — Use Hosted ZITADEL and Gateway-Owned Sessions

**Status:** Final  
**Date:** 2026-07-01  
**Decision Owners:** Product Owner, Security Owner, Technical Lead  
**Related:** FSD v1.0.1, Authorization Matrix v1.0.0, Security Threat Model v1.0.0

## Context

Terra Commerce requires authentication for global administrators, tenant staff, and tenant-scoped customers without placing a heavy identity platform on the initial 2 vCPU / 4 GB host.

## Decision

Use hosted ZITADEL as the identity provider and OpenID Connect Authorization Code with PKCE.

The Go gateway acts as a browser-facing backend-for-frontend and creates opaque server-side sessions. Browsers receive only a Secure, HttpOnly, SameSite cookie. Access and refresh tokens are not stored in browser local storage.

Identity organization:

- One platform organization for global privileged users.
- One ZITADEL organization per Terra Commerce tenant.
- Terra Commerce remains authoritative for tenant membership, tenant status, permissions, plan rules, support mode, and resource ownership.
- Customer identity is tenant scoped in Terra Commerce even when the same normalized email exists in another tenant.

Session rules:

- Session identifiers rotate after login and sensitive context changes.
- Password reset, user deactivation, tenant suspension, role-sensitive changes, and security incidents revoke affected sessions.
- MFA is required for global privileged roles before production.
- CSRF protection is mandatory for unsafe cookie-authenticated requests.
- Server-side refresh credentials are encrypted and rotatable.

## Alternatives Considered

- Custom password authentication: rejected because it increases security and maintenance risk.
- Self-hosted Keycloak: rejected for MVP due to resource and operational overhead.
- Browser-managed bearer tokens: rejected because token exposure risk and revocation handling are worse.

## Consequences

Positive: mature identity capabilities, lower VPS load, centralized MFA and recovery, and safer browser sessions.

Negative: external identity-provider dependency and synchronization between ZITADEL identities and Terra Commerce authorization records.

## Review Trigger

Review for provider cost, availability, data-residency requirements, or a future need to self-host identity.