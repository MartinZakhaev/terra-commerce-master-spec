# Document Dependency Map

**Document ID:** GOV-TC-004  
**Version:** 1.0.0  
**Status:** Approved

## 1. Purpose

This map defines which documents are upstream authorities and which downstream artifacts must be updated when an upstream decision changes.

## 2. Primary Dependency Chain

```text
PRD
  ↓
FSD + Business Rules + Use Cases
  ↓
System Architecture + ADRs
  ↓
Technical Design
  ↓
Database Design + Data Dictionary
  ↓
State Machines
  ↓
API + Event + SSE + Webhook Contracts
  ↓
Authorization Matrix + Security Controls
  ↓
Test Strategy + Acceptance Traceability
  ↓
Infrastructure + Operational Runbooks
  ↓
Application Implementation
```

## 3. Authority Rules

| Artifact | Must derive from | Must govern |
|---|---|---|
| PRD | Product strategy | FSD, scope, success metrics |
| FSD | PRD | use cases, business rules, acceptance criteria |
| Architecture | PRD, FSD, ADRs | service boundaries and data flow |
| Technical design | architecture, FSD | implementation approach |
| Database design | FSD, architecture, state machines | persistence implementation |
| State machines | FSD and business rules | valid transitions and API actions |
| API contracts | FSD, architecture, state machines, authorization | server and client integration |
| Event contracts | FSD, architecture, state machines | producers and consumers |
| Authorization matrix | roles, use cases, resources | gateway and backend enforcement |
| Test strategy | all approved requirements | release verification |
| Infrastructure | NFRs and architecture | deployment and operations |
| Code | all applicable approved artifacts | executable behavior |

## 4. Required Impact Updates

### PRD Change

Review and potentially update all downstream artifacts.

### FSD or Business Rule Change

Review architecture, technical design, state machines, data, contracts, authorization, tests, and code.

### State Change

Update state-machine document, FSD behavior, API operations, events, database status constraints, authorization, UI behavior, and tests.

### API Change

Update OpenAPI, affected FSD flow, authorization matrix, client implementations, integration tests, and release notes.

### Event Change

Update event contract, producers, consumers, SSE mapping where applicable, schema compatibility tests, and release notes.

### Data Model Change

Update database design, data dictionary, migrations, API or event schemas, retention rules, tests, and operational rollback guidance.

### Permission Change

Update authorization matrix, FSD actor flow, gateway and backend enforcement, UI visibility, audit rules, and tests.

## 5. Traceability Matrix

Each implementation-ready feature should be traceable as:

```text
PRD requirement
  → FSD requirement / business rule
  → use case and acceptance criteria
  → architecture component
  → data entities
  → state transitions
  → API operations and events
  → permissions
  → test cases
  → application PRs
```

## 6. Blocking Rules

Implementation is blocked when:

- The requirement has no approved or reviewable FSD behavior.
- An endpoint has no contract.
- A status-changing operation has no state transition definition.
- A protected operation has no permission mapping.
- Persistent data has no ownership or tenant-isolation definition.
- Acceptance criteria are not testable.

## 7. Current Baseline

The approved upstream baseline is:

`docs/01-product/prd/PRD_Terra_Commerce_v1.0.0_Final.md`

The next required downstream artifact is the Terra Commerce FSD v1.0.0 draft.