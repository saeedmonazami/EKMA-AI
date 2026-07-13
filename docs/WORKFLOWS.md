# Workflow Design Principles

## Status

No n8n workflows have been created. Workflow creation requires explicit project-owner approval.

## Intended workflow categories

- Appointment inquiry and availability lookup
- Appointment creation, rescheduling, and cancellation
- Patient and staff notifications
- Human handoff and exception management
- Data synchronization and reconciliation
- Operational monitoring and failure escalation

## Mandatory controls for any approved workflow

1. Define the trigger, inputs, outputs, owner, and user-facing message.
2. Validate and normalize inbound data at the boundary.
3. Use an idempotency strategy to prevent duplicate bookings or notifications.
4. Use explicit time-zone handling; default operational time zone is `Asia/Tehran` unless the approved use case says otherwise.
5. Confirm sensitive or irreversible actions before execution.
6. Route clinical, urgent, ambiguous, or identity-sensitive requests to human staff.
7. Record safe audit metadata and failures without logging secrets or unnecessary PHI.
8. Define retry behavior, alerting, manual recovery, and rollback.
9. Test with synthetic data and obtain approval before activation.

## Workflow documentation template

When workflows are approved, document each one with: purpose, owner, trigger, dependencies, data classification, permissions, happy path, failure paths, idempotency key, human handoff rules, observability, test evidence, and rollback plan.
