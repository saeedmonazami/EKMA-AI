# Database Strategy

## Status

PostgreSQL is a planned component. No database, schema, migration, or connection configuration has been implemented.

## Data ownership

| Data area | Initial location | Target system of record |
| --- | --- | --- |
| Appointment availability and calendar events | Google Calendar | Scheduling domain, with Calendar synchronization as approved |
| Temporary pilot register | Google Sheets | PostgreSQL |
| Patient/contact profile | Not yet implemented | PostgreSQL, subject to privacy and consent design |
| Conversations and handoffs | Not yet implemented | PostgreSQL or approved secure messaging store |
| Audit events | Not yet implemented | Append-oriented PostgreSQL audit store |

## Proposed logical domains

- Organizations and facilities
- Staff, roles, and permissions
- Patients or contacts, identifiers, and consent
- Services, practitioners, locations, and schedules
- Appointment requests, appointments, cancellations, and reschedules
- Conversations, messages, and handoffs
- Notifications and delivery attempts
- Payments and reconciliation references
- Knowledge documents and retrieval metadata
- Audit events and integration event history

## Design requirements before implementation

- Define the lawful basis, consent model, retention periods, and deletion workflows.
- Identify which fields are PHI or sensitive personal data and classify them.
- Use UUID or equivalent internal identifiers; do not use phone numbers or provider IDs as primary keys.
- Encrypt connections and protect backups; restrict access by role and environment.
- Include `created_at`, `updated_at`, provenance, and audit metadata where appropriate.
- Design every provider interaction for idempotency and reconciliation.
- Establish backup, restore, and recovery-point/recovery-time objectives before production data is stored.
