# ADR-0006 — Use Git Submodules for Application Repositories

**Status:** Final  
**Date:** 2026-06-30  
**Decision Owners:** Product Owner, Technical Lead  
**Related:** PRD v1.0.0, FSD v1.0.1, Governance v1.0.0

## Context

Terra Commerce uses a master specification repository as the authoritative documentation and release-composition source. Application source code is separated into repositories for the gateway, commerce core, superadmin web, tenant admin web, storefront web, and infrastructure.

The master repository must identify exactly which application revisions form a coordinated release without duplicating source code.

## Decision

Attach application repositories under root-level `apps/` using Git submodules.

Planned paths:

- `apps/gateway`
- `apps/commerce`
- `apps/superadmin-web`
- `apps/tenant-admin-web`
- `apps/storefront-web`
- `apps/infrastructure`

Each approved master-spec release pins every submodule to an explicit commit. The pinned commits form the authoritative release manifest together with release metadata.

## Alternatives Considered

### Monorepo containing all source code
Rejected because the chosen governance model separates authoritative specifications from independently maintained application repositories.

### Git subtree
Rejected because it copies application history and content into the master repository and makes synchronization less explicit.

### Documentation links only
Rejected because links do not create a reproducible, immutable release composition.

### Package-version manifest only
Useful but insufficient alone for repositories and infrastructure that are not distributed as packages.

## Consequences

### Positive

- Exact application commits are reproducible.
- Master-spec remains the release-composition authority.
- Application repositories retain independent history and access control.

### Negative

- Contributors must initialize and update submodules correctly.
- Detached HEAD behavior can confuse inexperienced users.
- Cross-repository changes require coordinated pull requests.

## Constraints

- Submodules must point to commits, not floating branch expectations.
- Application PRs must be linked from the master-spec change.
- The master-spec submodule update occurs only after application commits are ready.
- CI must initialize submodules recursively where needed.
- Secrets and local environment files must not be committed through submodules.

## Review Trigger

Review if repository count creates unacceptable coordination overhead, if a monorepo becomes operationally preferable, or if release tooling replaces submodules with an equally reproducible mechanism.
