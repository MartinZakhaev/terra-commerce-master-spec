# Operations

This directory contains the authoritative Terra Commerce deployment, recovery, incident-response, and production-support runbooks.

## Current authoritative runbooks

- [`Deployment_Rollback_Runbook_Terra_Commerce_v1.0.0_Final.md`](Deployment_Rollback_Runbook_Terra_Commerce_v1.0.0_Final.md) — release preflight, worker drain, migrations, point of no return, deployment, validation, and rollback.
- [`Backup_Restore_Disaster_Recovery_Runbook_Terra_Commerce_v1.0.0_Final.md`](Backup_Restore_Disaster_Recovery_Runbook_Terra_Commerce_v1.0.0_Final.md) — backup, isolated restore, split-brain fencing, provider reconciliation, disaster recovery, and validation.
- [`Incident_Response_Security_Incident_Runbook_Terra_Commerce_v1.0.0_Final.md`](Incident_Response_Security_Incident_Runbook_Terra_Commerce_v1.0.0_Final.md) — incident severity, containment, evidence, credential rotation, recovery gates, communication, and review.
- [`Operational_Support_Troubleshooting_Runbook_Terra_Commerce_v1.0.0_Final.md`](Operational_Support_Troubleshooting_Runbook_Terra_Commerce_v1.0.0_Final.md) — tenant support, jobs, outbox/Redis, SSE, providers, financial flows, inventory, reports, resources, certificates, secrets, and break-glass corrections.

## Historical drafts

- `Deployment_Rollback_Runbook_Terra_Commerce_v1.0.0_Draft.md`
- `Backup_Restore_Disaster_Recovery_Runbook_Terra_Commerce_v1.0.0_Draft.md`
- `Incident_Response_Security_Incident_Runbook_Terra_Commerce_v1.0.0_Draft.md`
- `Operational_Support_Troubleshooting_Runbook_Terra_Commerce_v1.0.0_Draft.md`

## Review authority

The suite audit and final verification are stored in [`../09-infrastructure/reviews/`](../09-infrastructure/reviews/).

## Downstream authority

These runbooks govern production changes, recovery exercises, incident handling, operator permissions, support procedures, evidence capture, and release-readiness operations.