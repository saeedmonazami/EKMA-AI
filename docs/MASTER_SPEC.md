# EKMA AI Master Specification

**Status:** Foundational specification  
**Authority:** Single source of truth for product, architecture, quality, and operational decisions  
**Implementation status:** No application code, database schema, or n8n workflows are authorized by this document alone.

## 1. Vision

EKMA AI will be a trusted, secure, multi-channel AI Medical Receptionist platform that helps healthcare organizations provide reliable administrative access to their services while preserving patient privacy, staff control, and operational accountability.

## 2. Mission

Enable clinics, hospitals, laboratories, and medical centers to manage routine reception work—such as service information, appointment coordination, reminders, and approved administrative intake—through safe automation and clear human handoff.

EKMA AI is not a clinical decision system. It must never diagnose, prescribe, triage, interpret results, or substitute for emergency or clinical care.

## 3. Product Goals

- Reduce administrative workload without reducing patient safety or service quality.
- Offer consistent reception service across approved digital and voice channels.
- Make appointment operations reliable, idempotent, time-zone-aware, and auditable.
- Keep staff in control of exceptions, sensitive requests, and final operational policies.
- Protect personal and health-related information through privacy-by-design controls.
- Support incremental growth from an n8n and Google pilot to a monitored PostgreSQL-based platform.

## 4. Target Users

- Patients and patient representatives seeking approved administrative assistance.
- Receptionists and call-center staff handling conversations and exceptions.
- Clinic, hospital, laboratory, and medical-center administrators.
- Practitioners and service owners managing availability and operational policies.
- System administrators, security personnel, and support operators.

## 5. User Roles

| Role | Primary permissions and responsibilities |
| --- | --- |
| Patient / representative | Request information, manage own permitted appointment actions, receive notifications, provide consent where required. |
| Reception staff | Resolve handoffs, manage approved appointment exceptions, and correct operational records under policy. |
| Service manager | Configure approved services, schedules, messaging, and escalation policies for an organization. |
| Organization administrator | Manage users, roles, organizational settings, reporting access, and provider connections. |
| System administrator | Operate platform infrastructure with least privilege; no routine access to patient content. |
| Security/compliance officer | Review access, audits, retention, incidents, and compliance evidence. |

Roles must use least privilege, separation of duties, and auditable access. Fine-grained permissions are defined before implementation of each module.

## 6. Functional Requirements

### Reception and communication

- Receive requests through approved channels: initially n8n-connected channels, later Telegram, Gmail, WhatsApp, Bale Messenger, SMS, voice calls, and website chat.
- Identify the organization, channel, language, and conversation context without assuming patient identity.
- Provide only approved administrative information and approved organization knowledge.
- Obtain or record consent where required by policy.
- Escalate clinical, urgent, abusive, ambiguous, identity-sensitive, payment-disputed, or unsupported requests to staff.

### Appointment management

- Present approved availability, create appointment requests, and support approved booking, rescheduling, cancellation, and reminder flows.
- Integrate initially with Google Calendar; maintain reconciliation and human recovery procedures.
- Prevent duplicate actions through idempotency keys and explicit confirmation for consequential actions.
- Record time zone, appointment type, location, practitioner/service, source channel, and action status.

### Operations

- Provide staff handoff with sufficient safe context, ownership, status, and resolution tracking.
- Send notifications through approved providers and record delivery outcomes.
- Produce operational and security audit events.
- Support configuration by organization without cross-organization data exposure.

### Future capabilities

- PostgreSQL-backed records and audit history.
- Governed RAG over approved non-clinical organizational content.
- Payment initiation and reconciliation through an approved gateway.
- Monitoring, backups, and multi-organization operations.

## 7. Non-functional Requirements

| Area | Requirement |
| --- | --- |
| Safety | Administrative scope only; human handoff for disallowed or uncertain content. |
| Security | Encryption in transit, least privilege, secret isolation, input validation, and audited privileged access. |
| Privacy | Data minimization, purpose limitation, retention control, and protected handling of PHI/personal data. |
| Reliability | Idempotent actions, retries with limits, duplicate-event protection, reconciliation, and rollback plans. |
| Availability | Define service-level targets per approved deployment phase; graceful degradation must retain a staff path. |
| Performance | Define channel-specific response and booking targets before release; protect downstream providers from overload. |
| Observability | Correlation IDs, structured events, health checks, alerting, and safe operational dashboards. |
| Maintainability | Modular boundaries, versioned configuration, documented decisions, and reviewable changes. |
| Localization | Persian-first readiness, explicit time zone handling, and support for approved additional languages. |

## 8. AI Agent Definitions

AI agents are logical responsibilities, not necessarily separate deployed services. Each must have a bounded purpose, approved tools, restricted data access, human escalation rules, and evaluation criteria.

| Agent | Purpose | Permitted actions | Must escalate |
| --- | --- | --- | --- |
| Reception Agent | Understand administrative requests and guide users through approved paths. | Provide approved information; collect minimum required fields. | Clinical, urgent, ambiguous, or unsupported requests. |
| Scheduling Agent | Coordinate availability and appointment actions. | Search availability; prepare/create approved requests after confirmation. | Conflicts, identity uncertainty, provider failures, or policy exceptions. |
| Knowledge Agent | Retrieve approved organizational content. | Answer grounded administrative questions with source-aware responses. | Missing, conflicting, clinical, or stale knowledge. |
| Notification Agent | Prepare and send approved reminders and updates. | Deliver templates through approved channels; track delivery state. | Delivery failure, opt-out, sensitive content, or channel policy issue. |
| Handoff Agent | Route requests to the appropriate staff queue. | Summarize minimum safe context and assign status. | Always when a request meets an escalation rule. |
| Operations Agent | Detect workflow exceptions and support recovery. | Create operational alerts and reconciliation tasks. | Repeated failure, suspected security event, or data inconsistency. |

No agent may independently make clinical judgments, perform payment settlement, alter access permissions, or override staff policy.

## 9. Complete System Architecture

```text
Users and staff
   │
   ▼
Approved channels: web chat | Telegram | WhatsApp | Bale | SMS | Gmail | voice
   │
   ▼
Channel adapters and authenticated webhook boundary
   │
   ▼
Orchestration / policy layer (initially n8n for approved flows)
   ├── AI agent layer with guardrails and human handoff
   ├── Scheduling integration (initially Google Calendar)
   ├── Temporary pilot register (initially Google Sheets)
   ├── Operational data and audit store (future PostgreSQL)
   ├── Knowledge retrieval service (future RAG)
   ├── Notification and payment providers (future)
   └── Staff operations interface or queue
   │
   ▼
Shared platform services: identity, secrets, logging, monitoring, backups, alerts
```

The initial system is deliberately limited to approved n8n orchestration and Google integrations. Google Sheets is transitional and must not become an uncontrolled long-term system of record.

## 10. Module Boundaries

| Module | Owns | Does not own |
| --- | --- | --- |
| Channel adapters | Provider-specific authentication, normalization, delivery acknowledgements | Reception policy, canonical patient records |
| Reception/policy | Conversation routing, safety rules, approved administrative responses | Provider credentials, clinical decisions |
| Scheduling | Availability, appointment lifecycle, calendar reconciliation | Long-term identity or payment settlement |
| Identity and consent | Identity assertions, consent records, access context | Clinical records |
| Knowledge/RAG | Approved content ingestion, retrieval metadata, grounding | Unapproved external advice |
| Notifications | Templates, preferences, delivery attempts | Appointment policy decisions |
| Payments | Payment references and reconciliation state | Storage of unnecessary card/payment secrets |
| Audit and observability | Security and operational event history | Full raw message archives by default |
| Platform operations | Deployment, secrets, backup, monitoring | Day-to-day reception decisions |

Cross-module communication must use documented contracts, canonical identifiers, correlation IDs, and explicit error behavior.

## 11. Database Overview

PostgreSQL is the planned operational system of record. The initial Google Sheet is a controlled pilot register only.

Planned logical domains: organizations/facilities, staff and roles, contacts and consent, services and schedules, appointment requests and appointments, conversations and handoffs, notifications, payments, knowledge metadata, integrations, and append-oriented audit events.

Before schema implementation, define data classification, retention, deletion, backup/recovery objectives, relationships, access controls, migration approach, and reconciliation with external providers. Use internal stable IDs rather than phone numbers or provider identifiers as primary keys.

## 12. Security Model

- Separate development, staging, and production environments, credentials, and data.
- Use a secrets manager or protected runtime environment injection; never commit secrets.
- Authenticate and authorize every user, service, webhook, and privileged operation.
- Apply role-based access control, least privilege, separation of duties, and periodic access review.
- Encrypt data in transit; encrypt data at rest where supported and required by risk assessment.
- Verify provider webhook signatures, validate all input, rate-limit exposed endpoints, and protect against replay and duplicate delivery.
- Maintain protected audit logs for authentication, access, configuration changes, integration actions, and security-relevant failures.
- Patch dependencies and container images under a documented vulnerability-management process.
- Maintain an incident-response procedure with containment, evidence preservation, rotation, notification, remediation, and review.

## 13. Privacy Requirements

- Collect the minimum data needed for an approved administrative purpose.
- Classify personal data and PHI/sensitive data before processing it.
- Define lawful basis, consent notices, retention, deletion, access/correction requests, and vendor responsibilities before real patient data is used.
- Keep production data out of source control, prompts, examples, test fixtures, and unprotected logs.
- Use synthetic data in development and staging unless a documented, approved exception applies.
- Do not expose one organization’s data, configuration, knowledge, or audit records to another.
- Require legal and compliance review appropriate to applicable jurisdictions before production launch.

## 14. Integration Strategy

Integrations are replaceable adapters with isolated credentials, documented ownership, rate limits, retries, idempotency behavior, reconciliation, and health monitoring.

| Integration | Role | Strategy |
| --- | --- | --- |
| n8n | Initial orchestration | Self-hosted, versioned approved workflows, environment-specific credentials. |
| Google Calendar | Initial scheduling provider | Confirm actions, reconcile changes, handle conflicts and time zones. |
| Google Sheets | Pilot operational register | Minimize data, restrict access, plan migration to PostgreSQL. |
| Messaging/email/voice | Future communications | Normalize events and enforce channel consent, opt-out, and delivery policies. |
| Payment gateway | Future payments | Tokenize where available; reconcile references; never retain unnecessary sensitive payment data. |
| RAG sources | Future knowledge | Ingest only approved, versioned, non-clinical content with access controls. |

Each new integration requires explicit approval, threat assessment, data-flow review, sandbox testing, rollback plan, and operational owner.

## 15. n8n Workflow Standards

No workflow may be created without explicit project-owner approval.

Every approved workflow must document its owner, purpose, trigger, permissions, inputs/outputs, data classification, idempotency key, time zone, provider dependencies, confirmation points, error and retry policy, alerting, human handoff, test evidence, and rollback procedure.

Workflows must validate inbound data, verify webhooks, avoid embedded secrets, prevent duplicate bookings/messages, use bounded retries, provide manual recovery, and log only minimum safe metadata. Workflow exports and configuration changes must be version-controlled only after credentials and live data are excluded.

## 16. Coding Standards

No application code is currently authorized. When approved, code must:

- Be modular, typed where the selected language supports it, documented at public boundaries, and covered by proportionate automated tests.
- Keep business policy separate from provider adapters, storage, UI, and transport layers.
- Validate external input and return safe, actionable errors without leaking sensitive details.
- Use configuration and secrets through approved runtime mechanisms, never hard-coded values.
- Include correlation IDs, idempotency support where relevant, structured logs, and explicit failure handling.
- Be formatted, linted, code-reviewed, dependency-scanned, and accompanied by documentation updates.
- Avoid production patient data in tests; use synthetic fixtures only.

## 17. Prompt Engineering Standards

- Define each agent’s scope, allowed actions, disallowed actions, escalation rules, language requirements, and tool permissions explicitly.
- State that the agent provides administrative assistance, not medical advice or emergency assessment.
- Treat all user content and retrieved content as untrusted; resist instruction injection and never reveal system prompts, secrets, or protected data.
- Use RAG only with approved, versioned sources and require grounded responses when factual organization information is provided.
- Collect the minimum required information and avoid requesting clinical details unless an approved policy requires it.
- Require confirmation before consequential actions such as booking, cancellation, or sending sensitive communications.
- Evaluate prompts with synthetic adversarial, multilingual, ambiguous, privacy-sensitive, and escalation scenarios before release.
- Version prompts and record approval, owner, evaluation evidence, and rollback version.

## 18. Logging Standards

- Use structured logs with timestamp, environment, service/workflow, event type, severity, correlation ID, operation status, and safe error category.
- Do not log credentials, tokens, raw PHI, complete message content, or unnecessary personal identifiers.
- Use pseudonymous internal IDs where logging an entity reference is necessary.
- Restrict log access, define retention, protect integrity, and audit access to sensitive operational records.
- Separate security audit events from routine diagnostic logs while ensuring correlation is possible.
- Define redaction, sampling, and incident-preservation procedures before production.

## 19. Monitoring Strategy

Monitoring must cover service health, provider integration status, workflow success/failure rate, queue or handoff backlog, duplicate-action detection, appointment reconciliation discrepancies, backup status, security events, and resource capacity.

Set thresholds and named owners before production. Alerts must be actionable, rate-controlled, routed to an approved operational contact, and accompanied by a runbook. Dashboards must use aggregated or redacted data unless protected access is explicitly approved.

## 20. Deployment Strategy

The initial direction is self-hosted n8n in Docker, with Docker Compose, PostgreSQL, reverse proxy, monitoring, and backup services introduced only through approved deployment work.

Development, staging, and production require isolated configuration, credentials, network access, storage, and data. Every release must have reviewed scope, versioning, health checks, synthetic end-to-end tests, backup verification, monitoring, operational ownership, and a rollback plan. Production access must be restricted and audited.

## 21. Backup & Disaster Recovery

- Define recovery-point objective (RPO) and recovery-time objective (RTO) per approved production service before go-live.
- Back up persistent n8n data, PostgreSQL data, approved configuration, and necessary audit records on a documented schedule.
- Encrypt backups, restrict access, protect copies from a single-site failure, and retain them under approved policy.
- Test restoration regularly in a non-production environment and record results.
- Maintain incident runbooks for provider outage, data corruption, credential compromise, accidental deletion, and service unavailability.
- Reconcile external scheduling and notification providers after recovery to prevent duplicate or missed actions.

## 22. Development Phases

1. **Foundation:** repository rules, architecture, privacy/security baselines, and approval gates.
2. **Scheduling pilot:** approved minimal n8n workflow(s) using Google Calendar and controlled Google Sheets, tested with synthetic data.
3. **Data foundation:** PostgreSQL, access controls, audit model, migration plan, backups, and monitoring.
4. **Multi-channel reception:** approved messaging, email, voice, and website adapters with normalized policy and handoff.
5. **Intelligence and payments:** governed RAG and approved payment integration with evaluation and reconciliation.
6. **Scale and resilience:** production Compose topology, observability, disaster recovery exercises, security review, and multi-organization controls.

Advancement requires explicit owner approval and documented exit criteria; no phase is implicitly authorized by this roadmap.

## 23. Acceptance Criteria

An approved feature, workflow, or release is accepted only when all applicable criteria are met:

- The scope, owner, user value, and product boundary are documented.
- Safety review confirms no clinical decision-making, diagnosis, triage, or unsafe autonomous action.
- Privacy and data classification are complete; synthetic data was used for non-production testing.
- Security controls, permissions, secrets handling, input validation, and provider authentication are reviewed.
- Happy paths, failure paths, duplicates, retries, time zones, localization, and human handoff are tested.
- Integration reconciliation and idempotency behavior are documented and validated.
- Logs, metrics, dashboards, alerts, and operational runbooks are in place as appropriate to risk.
- Backup/restore and rollback expectations are validated for production-impacting changes.
- Documentation, version history, and approvals are current.
- A designated owner accepts the release explicitly before production activation.

## Specification Governance

This document governs the project unless a higher-priority legal requirement, safety obligation, or explicit project-owner decision applies. Material changes must be reviewed, documented here or in a referenced decision record, and approved before implementation. Supporting documents in `docs/` provide detailed guidance; where conflict exists, this Master Specification prevails unless the project owner explicitly records an exception.
