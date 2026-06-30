# Terra Commerce Master Specification

Authoritative source of truth for Terra Commerce product, functional, architecture, data, API, event, security, infrastructure, quality, and operational specifications.

## Locked MVP baseline

- Multitenant SaaS e-commerce
- Go gateway
- Medusa.js v2 commerce engine
- PostgreSQL
- Redis Streams
- Server-Sent Events
- Biteship delivery integration
- One tenant = one store = one warehouse
- Initial target: 2 vCPU and 4 GB RAM

## Repository model

- Authoritative documents live under `docs/`.
- Application repositories live as Git submodules under `apps/`.
- A master-spec release pins each application to an explicit commit.
- Cross-application changes must update affected specifications and contracts.

## Approved product requirements

`docs/01-product/prd/PRD_Terra_Commerce_v1.0.0_Final.md`

## Planned application layout

```text
apps/
├── gateway/
├── commerce/
├── superadmin-web/
├── tenant-admin-web/
├── storefront-web/
└── infrastructure/
```

Do not duplicate application source code in this repository outside registered submodules.