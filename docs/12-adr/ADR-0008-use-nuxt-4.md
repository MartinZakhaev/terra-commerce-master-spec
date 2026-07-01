# ADR-0008 — Use Nuxt 4

**Status:** Final  
**Date:** 2026-07-01  
**Decision Owners:** Product Owner, Frontend Lead, Technical Lead

## Decision

Use Nuxt 4, Vue 3, and TypeScript for the storefront, tenant administration, and global administration web applications.

The storefront uses hybrid rendering. Administration applications use client rendering by default. Heavy browser components are lazy-loaded and all business APIs remain behind the Go gateway.

## Consequences

This provides one familiar frontend stack, fast delivery, storefront SEO, and controlled client bundles. Storefront SSR requires a Node runtime and performance budgets remain mandatory.