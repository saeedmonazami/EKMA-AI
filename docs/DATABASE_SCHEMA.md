# EKMA AI PostgreSQL ER Model

**Status:** Logical data-model specification only.  
**Implementation gate:** This document contains no SQL and does not authorize a database, migration, connection, or production data processing.

This is the proposed PostgreSQL operational data model for EKMA AI. It is subordinate to [MASTER_SPEC.md](MASTER_SPEC.md) and supplements [DATABASE.md](DATABASE.md). Field names are conceptual; physical names, data types, partitions, retention periods, encryption choices, and migrations require separate approved design work.

## Design Principles

- PostgreSQL becomes the operational system of record; Google Calendar remains an external scheduling provider and Google Sheets is only a controlled pilot register.
- Every tenant-owned record carries an `organization_id`; all tenant access must enforce that boundary.
- Use opaque internal UUID-style identifiers. Never use phone numbers, emails, payment references, or provider IDs as primary keys.
- Record `created_at`, `updated_at`, provenance/source, and actor context where applicable.
- Store minimum necessary personal data. Keep sensitive values out of ordinary logs and analytical datasets.
- Use state transitions, append-oriented history, idempotency records, and reconciliation records instead of silently overwriting consequential events.
- Clinical records, diagnosis, treatment, test interpretation, and unnecessary payment secrets are outside this model.

## Entity Relationship Overview

```text
Organization ──< Facility ──< Location
      │              │
      ├──< OrganizationMember >── UserAccount ──< UserRole >── Role ──< RolePermission >── Permission
      ├──< Service ──< ServiceSchedule >── Practitioner
      ├──< Contact ──< ContactIdentifier
      │       ├──< ConsentRecord
      │       ├──< CommunicationPreference
      │       ├──< AppointmentRequest ──< Appointment >── Service / Practitioner / Location
      │       ├──< Conversation ──< ConversationParticipant / Message
      │       └──< HandoffCase
      ├──< Notification ──< NotificationDelivery
      ├──< PaymentRequest ──< PaymentTransaction
      ├──< KnowledgeDocument ──< KnowledgeDocumentVersion
      ├──< IntegrationConnection ──< ExternalReference / IntegrationEvent
      └──< AuditEvent
```

`<` denotes “one-to-many.” Many-to-many relationships use explicit junction tables so that permissions, provenance, and history can be audited.

## Shared Entity Conventions

| Convention | Requirement |
| --- | --- |
| Identifier | Internal UUID-style primary key on every table unless a controlled append-only event identifier is more appropriate. |
| Tenant scope | `organization_id` is mandatory on tenant-owned rows and part of every access-policy predicate. |
| Lifecycle | Use explicit status/state fields and timestamps; avoid hard deletion of operational records until retention rules permit it. |
| Provenance | Capture source system, source event/reference, correlation ID, and actor type where relevant. |
| Sensitive data | Mark fields by classification: operational, personal, sensitive/PHI, financial reference, or security metadata. |
| External IDs | Store provider identifiers in `external_references`; do not embed provider-specific identifiers throughout domain tables. |
| Time | Store instants in a timezone-aware form and retain the user/organization time zone used for scheduling decisions. |

## Tables by Domain

### 1. Tenant, facility, and configuration

| Table | Purpose | Key logical attributes | Relationships |
| --- | --- | --- | --- |
| `organizations` | Tenant and healthcare organization boundary. | ID, legal/display name, status, default locale/time zone, data-policy version. | Parent of most tenant-owned entities. |
| `facilities` | Clinic, hospital, laboratory, or unit under an organization. | ID, organization ID, name, status, contact policy. | Belongs to organization; parent of locations/services. |
| `locations` | Bookable/communicated physical or virtual locations. | ID, organization/facility IDs, name, address fields, time zone, status. | Referenced by services, schedules, appointments. |
| `organization_settings` | Versioned non-secret operational settings. | ID, organization ID, setting key, approved value reference, version, effective period. | Belongs to organization; changes are audited. |
| `service_categories` | Grouping of administrative services. | ID, organization ID, name, status, display order. | Parent of services. |
| `services` | Bookable or informational administrative service definition. | ID, organization/facility/category IDs, name, duration, status, booking policy reference. | Referenced by schedules, appointment requests, appointments, fees. |
| `service_locations` | Allows a service at many locations. | Service ID, location ID, status, local policy override. | Junction: services ↔ locations. |

### 2. Identity, access, and staff

| Table | Purpose | Key logical attributes | Relationships |
| --- | --- | --- | --- |
| `user_accounts` | Platform account for staff/administrators; may be external-identity backed. | ID, identity provider subject/reference, status, last-authentication metadata. | Linked to organization memberships and roles. |
| `organization_members` | Scoped staff membership in one organization. | ID, organization ID, user account ID, staff display name, status, employment/reference metadata. | Junction: organizations ↔ user accounts. |
| `roles` | Named permissions bundle, system or tenant scoped. | ID, organization scope nullable for system role, name, status. | Linked through user roles and role permissions. |
| `permissions` | Atomic named capability. | ID, capability key, domain, sensitivity tier, description. | Linked through role permissions. |
| `role_permissions` | Permission assignment to a role. | Role ID, permission ID, constraint scope. | Junction: roles ↔ permissions. |
| `user_roles` | Role assignment to a member. | Organization member ID, role ID, grantor, effective period, status. | Junction: members ↔ roles. |
| `access_review_cases` | Periodic or event-driven access review. | ID, organization ID, subject, reviewer, due date, status, outcome. | References membership/role assignments; audited. |

### 3. Contacts, consent, and communication preferences

| Table | Purpose | Key logical attributes | Relationships |
| --- | --- | --- | --- |
| `contacts` | Minimum necessary patient/representative contact record. | ID, organization ID, display/preferred name, status, preferred locale, data classification. | Parent of identifiers, consent, preferences, appointments, conversations. |
| `contact_identifiers` | Protected contact channels/identifiers. | ID, contact ID, identifier type, normalized protected value/reference, verification state, primary flag. | Many per contact; uniqueness scoped by organization/type. |
| `contact_relationships` | Authorized representative relationship. | ID, organization ID, subject contact ID, related contact ID, relationship type, verification/effective period. | Self-referential contacts relationship. |
| `consent_records` | Immutable evidence of consent or withdrawal. | ID, organization/contact IDs, purpose, channel/scope, policy/notice version, state, captured time/source. | Many per contact; drives communication/profile workflows. |
| `communication_preferences` | Current allowed contact and channel choices. | ID, contact/organization IDs, channel, preference state, locale, quiet hours/time zone. | Many per contact/channel; derived from consent/policy. |
| `privacy_request_cases` | Data-subject request lifecycle. | ID, organization/contact IDs, request type, identity-assurance level, due date, status, resolution. | References consent and audit events. |

### 4. Scheduling

| Table | Purpose | Key logical attributes | Relationships |
| --- | --- | --- | --- |
| `practitioners` | Administrative representation of a service provider; not a clinical record. | ID, organization/facility IDs, display name, status, booking visibility. | Referenced by schedules and appointments. |
| `service_practitioners` | Allowed practitioner-service pairing. | Service ID, practitioner ID, status, policy override. | Junction: services ↔ practitioners. |
| `service_schedules` | Availability rule or imported schedule window. | ID, organization/service/practitioner/location IDs, recurrence/rule reference, start/end, time zone, status. | Feeds availability; linked to external calendar references. |
| `appointment_requests` | Requested booking intent before final appointment outcome. | ID, organization/contact/service IDs, requested windows, channel, status, correlation/idempotency reference. | May create/modify one or more appointment attempts. |
| `appointments` | Canonical administrative appointment state. | ID, organization/contact/service/practitioner/location IDs, start/end, time zone, status, confirmation state, source. | Has history, external references, reminders, payments. |
| `appointment_participants` | Additional authorized attendees/representatives. | Appointment ID, contact ID, participant role, status. | Junction: appointments ↔ contacts. |
| `appointment_status_history` | Append-only appointment lifecycle history. | ID, appointment ID, prior/new status, reason category, actor/source, occurred time. | Many per appointment. |
| `waitlist_entries` | Controlled request for a future suitable slot. | ID, organization/contact/service IDs, criteria, status, expiration. | May lead to appointment request/notification. |

### 5. Conversations and handoffs

| Table | Purpose | Key logical attributes | Relationships |
| --- | --- | --- | --- |
| `conversations` | Channel-scoped interaction thread. | ID, organization ID, channel, status, locale, correlation root, retention class. | Parent of participants, messages, handoffs. |
| `conversation_participants` | Contact/staff/system participation in a conversation. | Conversation ID, participant type, contact/member reference, join/leave state. | Junction to contacts/members. |
| `messages` | Minimum retained inbound/outbound message metadata and approved content reference. | ID, conversation ID, direction, sender type, channel message reference, content classification, status, occurred time. | May link to notifications, agent decisions, external references. |
| `handoff_cases` | Staff-owned exception and escalation work item. | ID, organization/conversation/contact IDs, reason, priority, queue, owner, status, due time. | Has handoff history and safe context reference. |
| `handoff_case_history` | Append-only status/assignment/resolution trail. | ID, handoff case ID, event type, actor/source, occurred time, safe note reference. | Many per handoff case. |
| `agent_decision_records` | Safe structured outcome of an agent decision. | ID, organization/conversation IDs, agent type/version, decision type, confidence band, handoff reason, correlation ID. | Linked to messages, handoffs, audit events; not hidden reasoning. |

### 6. Notifications

| Table | Purpose | Key logical attributes | Relationships |
| --- | --- | --- | --- |
| `notification_templates` | Approved versioned message template metadata. | ID, organization ID, purpose, channel, locale, version, approval/status. | Referenced by notifications. |
| `notifications` | Requested transactional communication. | ID, organization/contact IDs, template ID, purpose, channel, scheduled time, status, idempotency key. | Has delivery attempts; may reference appointment/payment. |
| `notification_deliveries` | Provider delivery attempt and receipt lifecycle. | ID, notification ID, attempt number, provider reference, status, occurred time, safe error category. | Many per notification. |
| `notification_suppressions` | Record why a send was prevented. | ID, organization/contact IDs, channel/purpose, reason, source, occurred time. | Supports consent and compliance audit. |

### 7. Billing and payments

| Table | Purpose | Key logical attributes | Relationships |
| --- | --- | --- | --- |
| `fee_schedules` | Approved administrative fee definitions. | ID, organization/service IDs, currency, amount, effective period, status, approval reference. | Referenced by payment requests. |
| `payment_requests` | Intent to collect a permitted payment. | ID, organization/contact/appointment IDs, fee schedule ID, amount/currency, status, idempotency key. | Has transactions; no card data. |
| `payment_transactions` | Gateway-confirmed attempt/status record. | ID, payment request ID, provider reference, status, amount/currency, occurred time, reconciliation state. | Many per payment request. |
| `payment_status_history` | Append-only billing/payment state transitions. | ID, payment request/transaction IDs, prior/new status, source, occurred time. | Supports reconciliation and dispute handling. |
| `billing_cases` | Staff review for dispute, refund, duplicate, or exception. | ID, organization/contact/payment request IDs, case type, status, owner, outcome. | Linked to payment history and audit. |

### 8. Knowledge and RAG governance

| Table | Purpose | Key logical attributes | Relationships |
| --- | --- | --- | --- |
| `knowledge_documents` | Approved organizational knowledge document identity. | ID, organization ID, title, category, owner, status, retention class. | Parent of versions. |
| `knowledge_document_versions` | Immutable approved source version. | ID, document ID, version, source URI/reference, checksum, approval state, effective period. | May have retrieval chunks. |
| `knowledge_chunks` | Retrieval units and metadata, not unrestricted external content. | ID, document version ID, sequence, content reference, classification, embedding reference/version. | Many per document version. |
| `knowledge_access_policies` | Role/organization restrictions on knowledge use. | ID, document/version ID, audience, policy state, effective period. | Controls retrieval visibility. |
| `knowledge_review_cases` | Content-gap, stale, conflicting, or approval review. | ID, organization/document IDs, reason, priority, owner, status. | Can be initiated by FAQ workflow. |

### 9. Integrations, operations, and idempotency

| Table | Purpose | Key logical attributes | Relationships |
| --- | --- | --- | --- |
| `integration_connections` | Metadata for an approved provider connection; no raw secrets. | ID, organization ID, provider/type, environment, status, credential reference, health state. | Parent of external references/events. |
| `external_references` | Maps a canonical entity to an external provider identity. | ID, organization/integration IDs, entity type/ID, external type/value reference, status, last-synced time. | Used by calendars, messages, payments, etc. |
| `integration_events` | Append-oriented normalized inbound/outbound provider event metadata. | ID, organization/integration IDs, event type, external event reference, correlation ID, processing status, occurred time. | Supports replay protection/reconciliation. |
| `idempotency_records` | Prevents repeated consequential actions. | ID, organization ID, operation type, idempotency key, request fingerprint, result reference, expiry/status. | Referenced by appointments, payments, notifications. |
| `reconciliation_cases` | Tracks provider/internal state mismatch. | ID, organization/integration/entity reference, mismatch type, priority, status, owner. | Linked to integration and audit history. |
| `operational_incidents` | Controlled operations/security incident record. | ID, organization nullable, severity, category, status, owner, timestamps. | Linked to alerts, audit events, remediation tasks. |

### 10. Audit, history, and retention

| Table | Purpose | Key logical attributes | Relationships |
| --- | --- | --- | --- |
| `audit_events` | Append-only material security and business-action evidence. | ID, organization ID, actor type/reference, action, entity type/ID, correlation ID, outcome, occurred time, safe metadata. | References any domain entity logically; protected access. |
| `data_change_history` | Field-level or approved snapshot-delta history for sensitive entities. | ID, organization ID, entity type/ID, version, change type, actor/source, occurred time, protected delta reference. | Used where audit detail exceeds status history. |
| `retention_policies` | Versioned retention and deletion rules by data class/entity. | ID, organization scope nullable, data class/entity type, retention period, legal-hold behavior, version. | Drives retention cases. |
| `retention_execution_records` | Evidence of retention, deletion, anonymization, or hold action. | ID, policy ID, entity reference, action, outcome, occurred time, operator/source. | Audited; never silently destroys evidence. |
| `legal_holds` | Prevent deletion/alteration under authorized hold. | ID, organization ID, scope/entity reference, reason category, effective period, status. | Overrides retention execution. |

## Relationship Rules

- An organization has many facilities, services, contacts, members, integrations, and audit events; no tenant-owned entity can belong to more than one organization.
- A facility has many locations and may have many services; services may be offered at multiple locations and by multiple practitioners through junction tables.
- A contact may have multiple protected identifiers, consent records, preferences, conversations, appointment requests, appointments, and handoff cases.
- An appointment belongs to exactly one organization, contact, and service; practitioner and location are required or optional according to the approved service policy, never inferred silently.
- Appointment status changes are immutable events in `appointment_status_history`; the current appointment state is a materialized operational view of that history.
- A conversation has many participants and messages; a message belongs to one conversation and may be associated with an agent decision record or notification delivery.
- A notification belongs to one contact and organization; it uses one approved template version and can have many delivery attempts, but exactly one terminal delivery/suppression outcome.
- A payment request may relate to one appointment and has many provider transaction attempts; gateway references are external references, not payment credentials.
- An external reference maps one canonical entity to a provider identity. Uniqueness is enforced per integration, external entity type, and external reference.
- Audit events reference entities polymorphically by entity type and ID; they are never cascade-deleted with the referenced row.

## Index Model

Indexes are logical requirements; exact index type and physical design require workload validation.

| Table/domain | Required index or access path | Reason |
| --- | --- | --- |
| All tenant-owned tables | `organization_id` plus common status/time filters. | Tenant isolation and scoped operations. |
| `contacts` / `contact_identifiers` | Organization, identifier type, normalized protected lookup representation; contact ID. | Verified contact lookup and duplicate detection without broad scans. |
| `consent_records` / `communication_preferences` | Contact, organization, purpose/channel, current state, effective time. | Consent/opt-out enforcement. |
| `appointments` | Organization, start time, status; contact and start time; practitioner/location and time range; external reference mapping. | Availability, patient lookup, schedule operations, reconciliation. |
| `appointment_requests` / histories | Organization, status, created time; correlation ID; idempotency reference. | Work queues, traceability, duplicate prevention. |
| `service_schedules` | Organization, service/practitioner/location, active time range. | Availability computation. |
| `conversations` / `messages` | Organization, channel, status, last activity; conversation and occurred time; provider message reference. | Routing, history retrieval, duplicate provider event detection. |
| `handoff_cases` | Organization, queue/owner, priority, status, due time. | Staff queue and SLA monitoring. |
| `notifications` / deliveries | Organization, scheduled time/status; contact/purpose; idempotency key; provider delivery reference. | Dispatch, suppression, delivery reconciliation. |
| Payments | Organization/status/time; payment request; provider reference; idempotency key. | Reconciliation and dispute processing. |
| `integration_events` / `external_references` | Integration + external reference; processing status + occurred time; entity reference. | Replay protection and reconciliation. |
| `idempotency_records` | Organization + operation type + idempotency key (unique). | Exactly-once effect protection. |
| `audit_events` | Organization + occurred time; entity type/ID + occurred time; correlation ID; actor reference. | Investigations and compliance queries. |
| History/event tables | Parent entity + occurred time; organization + occurred time. | Timeline reconstruction and retention. |

Sensitive identifier indexes must use an approved protected search representation; raw identifiers must not be broadly indexable or exposed through reporting views.

## Constraints and Integrity Rules

### Primary, foreign, and uniqueness constraints

- Every table has a stable primary key; every required parent relationship has an enforced foreign key.
- A child’s `organization_id` must match its parent entity’s organization; enforce with composite tenant-aware keys or equivalent guardrails.
- A contact may have at most one active preferred identifier per identifier type, subject to approved policy.
- A contact may have at most one current communication preference per channel/purpose combination.
- A notification must have one approved template version, one intended contact, and one organization-matching recipient.
- One active idempotency record exists per organization, operation type, and idempotency key.
- Provider event and external-reference uniqueness is scoped to the integration connection and entity type.
- Organization role assignments are unique for the same member, role, and overlapping effective period unless a documented exception is approved.

### Domain constraints

- All appointment start/end times are valid, end follows start, and the stored decision time zone is valid.
- Appointment state transitions follow an approved state machine; cancellation/reschedule reasons are recorded where policy requires.
- A confirmed appointment must have a provider reconciliation state that is confirmed or explicitly pending review; timeout is never treated as success.
- Notification delivery cannot transition from a terminal suppressed/failed state to sent without a new approved notification action.
- Payment amount/currency must match the approved fee schedule or a separately approved exception; transaction success requires provider confirmation and reconciliation.
- Consent withdrawal is effective no later than the next outbound communication decision; legal retention obligations may preserve evidence without permitting continued messaging.
- Knowledge retrieval is allowed only for an approved, effective document version and access policy.

### Delete and retention constraints

- No cascade delete from organizations, contacts, appointments, payments, audit events, or history tables.
- Deactivation/soft delete is the default for operational records. Anonymization or physical deletion occurs only via approved retention execution and must honor legal holds.
- Audit and history data are append-only to ordinary roles; corrections are new compensating events, not destructive edits.

## Audit and History Model

### What is audited

Audit events are required for authentication and authorization decisions; profile/consent changes; appointment, notification, payment, and integration actions; workflow execution outcomes; agent decision types and escalations; configuration changes; privileged access; security alerts; retention actions; and staff handoff lifecycle events.

### Audit-event content

An audit event records who/what acted, organization scope, action, target entity reference, time, source/channel, correlation ID, outcome, and safe metadata. It must not include secrets, raw payment data, or unnecessary raw message/PHI content. Content references, where justified, are access-controlled separately.

### Entity history

Use specialized status-history tables for appointments, payments, handoffs, and delivery attempts. Use `data_change_history` for sensitive field-level change evidence, with protected deltas and version sequencing. Every correction creates a new record that references the prior version.

### Tamper resistance and retention

Audit/history tables are append-only for application roles, protected from routine administrative mutation, and backed up with integrity verification. Access is itself audited. Retention periods, legal holds, and deletion evidence are governed by `retention_policies`, `legal_holds`, and `retention_execution_records`.

## Permissions Model

### Database access roles

| Database role | Allowed scope | Prohibited scope |
| --- | --- | --- |
| Migration owner (future) | Approved schema changes in controlled deployment. | Runtime business operations; routine data access. |
| Application runtime | Minimum CRUD per module through approved views/procedures or scoped tables. | Schema changes, unrestricted cross-tenant queries, audit mutation. |
| n8n integration runtime | Only approved workflow data paths and idempotency/integration records. | Broad patient lookup, role changes, direct audit alteration. |
| Reporting runtime | Approved aggregated/de-identified views by organization and role. | Raw sensitive identifiers and cross-tenant detail. |
| Operations read-only | Health/reconciliation metadata and redacted operational views. | Sensitive content and write access. |
| Security/compliance reader | Protected audit/history and approved investigation views. | Routine operational mutation, secret retrieval. |
| Backup/restore operator | Backup duties under controlled process. | Routine application queries or role administration. |

### Authorization enforcement

- Use separate database identities per environment and workload; shared administrator credentials are prohibited.
- Enforce tenant isolation with PostgreSQL row-level security or an equivalent database-enforced strategy for every tenant-owned table and view.
- Use approved organization and actor context only after authentication; never trust a client-provided tenant ID without server-side validation.
- Expose least-privilege views for reporting and operations; sensitive identifier values and protected content require dedicated, audited access paths.
- Separate role administration, data access, and audit review responsibilities. Privileged changes require approval and audit events.
- Database access never grants permission for clinical actions; such actions are outside EKMA AI’s scope.

## Implementation Prerequisites

Before physical schema work is approved, finalize: applicable legal/privacy obligations; data classification; tenant model; consent and representative policy; retention schedule; appointment state machine; identity-verification levels; encryption/key-management design; reporting views; RPO/RTO; backup/restore tests; migration strategy; performance workload; row-level security design; and a field-by-field data dictionary.
