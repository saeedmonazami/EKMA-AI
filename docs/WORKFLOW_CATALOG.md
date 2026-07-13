# EKMA AI Workflow Catalog

**Status:** Design catalog only. No n8n workflows, workflow exports, code, integrations, or credentials are created by this document.  
**Approval gate:** Every catalog item remains `Proposed` until the project owner explicitly approves its individual implementation scope.

## Catalog Rules

- All workflows must comply with [MASTER_SPEC.md](MASTER_SPEC.md) and [WORKFLOWS.md](WORKFLOWS.md).
- “Dependency” identifies required approved systems, policies, agents, or upstream workflows; it does not authorize them.
- Priorities: **P0** = safety/security/platform-critical; **P1** = core pilot capability; **P2** = planned capability; **P3** = future optimization.
- Every implemented workflow must define an owner, idempotency key where consequential, time zone, data classification, access policy, retries, alerts, audit events, manual recovery, tests using synthetic data, and rollback.
- All patient-facing workflows are administrative only. Clinical, urgent, ambiguous, or identity-sensitive requests must hand off to authorized staff.

## Intake, Identity, and Handoff

| ID & workflow | Purpose | Trigger | Inputs | Outputs | Dependencies | Priority | Status |
| --- | --- | --- | --- | --- | --- | --- | --- |
| WF-001 Patient Registration & Appointment Booking | Pilot workflow: register minimum contact data, offer availability, confirm a slot, create Calendar event, and save a Sheets record. | Authenticated `register` or `confirm` booking request. | Minimum contact fields, approved doctor/calendar, date/slot, confirmation, idempotency key. | Pending offer or confirmed appointment reference; safe audit state. | n8n, Google Calendar, Google Sheets, approved webhook channel. | P1 | Designed — awaiting JSON approval |
| WF-002 Identity Verification | Establish sufficient identity confidence for protected actions. | Protected action or identity mismatch. | Identity assertions, verification method, policy. | Verified level, failure, or staff task. | Identity policy, staff queue, future identity provider. | P0 | Proposed |
| WF-003 Consent Capture | Record required communication/data-processing consent. | Registration, preference change, or policy-required interaction. | Consent notice version, response, identity context. | Consent state and audit event. | Consent policy, profile store. | P0 | Proposed |
| WF-004 Consent Withdrawal | Apply a consent withdrawal or opt-out request. | Patient/representative request. | Identity level, consent scope, request. | Updated consent state; suppression tasks. | Profile store, Notification Agent, audit store. | P0 | Proposed |
| WF-005 Channel Message Intake | Authenticate, validate, normalize, and correlate inbound channel events. | Approved inbound webhook/message. | Provider payload, signature, channel metadata. | Normalized event or rejected event. | Channel adapter, n8n, security policy. | P0 | Proposed |
| WF-006 Reception Intent Routing | Route an administrative request to the correct agent/workflow. | Normalized inbound request. | Request text/voice transcript, context, organization scope. | Route, safe response, or handoff. | Reception Agent, n8n, staff queue. | P1 | Proposed |
| WF-007 Human Staff Handoff | Create, assign, and track an exception requiring staff action. | Agent/policy escalation or staff request. | Correlation ID, safe summary, reason, priority. | Staff task, acknowledgement, status. | Staff queue, audit store, Notification Agent. | P0 | Proposed |
| WF-008 Handoff Resolution | Close a staff task and safely notify the requester where appropriate. | Staff marks task resolved. | Resolution status, approved response, requester context. | Closed task; optional notification request. | Staff queue, Notification Agent, audit store. | P1 | Proposed |
| WF-009 Data Subject Request Intake | Capture an access, correction, deletion, or privacy request. | Authenticated privacy request. | Identity level, request type, scope. | Case record and compliance handoff. | Privacy policy, authorized staff queue, future profile store. | P0 | Proposed |
| WF-010 Emergency/Clinical Escalation | Safely route disallowed clinical or potentially urgent content. | Clinical/urgent detection or explicit user request. | Normalized message, channel, locale. | Immediate safe response; priority staff handoff. | Reception/FAQ/Voice Agent, staff policy. | P0 | Proposed |

## Scheduling and Calendar

| ID & workflow | Purpose | Trigger | Inputs | Outputs | Dependencies | Priority | Status |
| --- | --- | --- | --- | --- | --- | --- | --- |
| WF-011 Availability Lookup | Return approved available appointment options. | Availability inquiry. | Service, practitioner/location preference, date range, time zone. | Available slots or safe exception. | Appointment Agent, Calendar Agent, Google Calendar. | P1 | Proposed |
| WF-012 Book Appointment | Create an appointment after validation and confirmation. | Confirmed booking request. | Identity level, chosen slot, service, idempotency key. | Confirmed/pending appointment result. | WF-011, Calendar Agent, Google Calendar, pilot register. | P1 | Proposed |
| WF-013 Appointment Confirmation | Deliver a confirmed booking summary through an approved channel. | Successful booking/reconciliation. | Appointment reference, consent, template, channel preference. | Delivery result or staff exception. | WF-012, Notification Agent. | P1 | Proposed |
| WF-014 Reschedule Appointment | Change an existing appointment under policy. | Confirmed reschedule request. | Appointment reference, new slot, identity level, idempotency key. | Updated appointment or conflict/handoff. | WF-011, Calendar Agent, Google Calendar. | P1 | Proposed |
| WF-015 Cancel Appointment | Cancel an appointment under policy. | Confirmed cancellation request. | Appointment reference, identity level, cancellation reason if required. | Cancellation result; notification request. | Calendar Agent, Google Calendar, Notification Agent. | P1 | Proposed |
| WF-016 Appointment Status Lookup | Provide permitted status/details of an appointment. | Patient/staff inquiry. | Identity level, appointment reference, organization scope. | Safe appointment status or handoff. | Calendar Agent, future scheduling store. | P1 | Proposed |
| WF-017 Calendar Change Reconciliation | Detect and resolve differences with calendar provider state. | Scheduled run or provider change event. | Provider event, internal/pilot record, correlation data. | Reconciled state, exception, or staff task. | Google Calendar, Calendar Agent, audit store. | P0 | Proposed |
| WF-018 Booking Conflict Resolution | Handle slot conflicts, duplicate booking risk, or unknown outcomes. | Conflict/error from booking workflow. | Requested action, provider state, idempotency key. | Safe alternative, reconciliation task, or handoff. | WF-012/WF-014, Calendar Agent, staff queue. | P0 | Proposed |
| WF-019 Waitlist Management | Maintain approved requests for newly available slots. | Waitlist request or freed slot. | Service criteria, contact consent, availability event. | Waitlist state; offer request or staff task. | Future scheduling store, Notification Agent, Calendar Agent. | P2 | Proposed |
| WF-020 No-show Follow-up | Create approved administrative follow-up after a no-show. | Staff/provider no-show status. | Appointment reference, policy, consent, channel preference. | Follow-up notification or staff task. | Scheduling store, Reminder/Notification Agent. | P2 | Proposed |

## Knowledge, Communication, and Notifications

| ID & workflow | Purpose | Trigger | Inputs | Outputs | Dependencies | Priority | Status |
| --- | --- | --- | --- | --- | --- | --- | --- |
| WF-021 Administrative FAQ Response | Answer a grounded organizational FAQ. | FAQ request. | Question, organization, locale, approved content. | Approved answer, clarification, or handoff. | Medical FAQ Agent, future RAG/content store. | P1 | Proposed |
| WF-022 Knowledge Content Gap | Record unanswered, stale, or conflicting knowledge content. | FAQ Agent cannot ground an answer. | Question category, source status, safe metadata. | Content-review task. | Medical FAQ Agent, content owner queue. | P2 | Proposed |
| WF-023 Appointment Reminder Scheduling | Decide and queue an approved future reminder. | Appointment creation/update or scheduled evaluation. | Appointment status, reminder policy, consent, time zone. | Scheduled/suppressed reminder request. | Reminder Agent, WF-012/WF-014, preference store. | P1 | Proposed |
| WF-024 Appointment Reminder Delivery | Send a due appointment reminder. | Due reminder event. | Template, recipient reference, channel preference, idempotency key. | Delivery status or failure task. | Notification Agent, approved channel adapter. | P1 | Proposed |
| WF-025 Notification Delivery Reconciliation | Reconcile provider acknowledgements, bounces, and retries. | Provider callback or scheduled run. | Delivery receipt, message reference, provider state. | Final delivery status, retry, or staff task. | Notification Agent, channel adapter, audit store. | P1 | Proposed |
| WF-026 Notification Opt-out Enforcement | Prevent messaging where consent or opt-out prohibits it. | Any outbound notification request. | Recipient reference, channel, consent/preference state. | Approved send or suppression record. | WF-004, Notification Agent, profile store. | P0 | Proposed |
| WF-027 Service Change Notification | Notify affected contacts of an approved schedule/service change. | Staff-approved change event. | Change details, affected appointment references, templates. | Delivery requests and outcomes. | Scheduling source, Notification Agent, staff approval. | P1 | Proposed |
| WF-028 Website Chat Session Intake | Normalize and safely route web-chat sessions. | Approved website chat event. | Session metadata, message, consent notice state. | Normalized request, response, or handoff. | Future web chat adapter, WF-005/WF-006. | P2 | Proposed |
| WF-029 Email Reception Intake | Normalize and route approved inbound email. | Approved Gmail event. | Sender, message, attachment metadata, authorization state. | Routed request, safe reply request, or handoff. | Future Gmail adapter, WF-005/WF-006. | P2 | Proposed |
| WF-030 Messaging Channel Intake | Normalize Telegram/WhatsApp/Bale/SMS messages. | Provider event. | Signed provider payload, channel metadata, message. | Normalized event or rejection. | Future channel adapters, WF-005. | P2 | Proposed |

## Voice, Profile, and Payments

| ID & workflow | Purpose | Trigger | Inputs | Outputs | Dependencies | Priority | Status |
| --- | --- | --- | --- | --- | --- | --- | --- |
| WF-031 Voice Call Intake | Safely receive and route an approved voice call event. | Telephony provider call event. | Caller metadata, consent notice state, transcript/intent. | Response, transfer request, or handoff. | Future telephony adapter, Voice Agent, WF-006. | P2 | Proposed |
| WF-032 Voice-to-Staff Transfer | Transfer or create a callback task for a voice caller. | Voice Agent escalation or caller request. | Safe call summary, routing criteria, callback details. | Transfer/callback task and status. | Voice Agent, staff call queue, Notification Agent. | P1 | Proposed |
| WF-033 Patient Profile Update | Validate and submit permitted contact/preference updates. | Authenticated update request. | Identity level, changed fields, consent/policy. | Updated/pending profile or handoff. | Patient Profile Agent, pilot register/future PostgreSQL. | P1 | Proposed |
| WF-034 Duplicate Profile Review | Detect possible duplicate profiles and route safe review. | Registration/update match signal. | Minimum matching metadata, confidence, policy. | Review task; no automatic merge. | Patient Profile Agent, authorized staff queue. | P0 | Proposed |
| WF-035 Communication Preference Update | Update allowed channels, language, and notification choices. | Patient/staff-approved request. | Identity level, preference choices, consent. | Updated preferences and audit event. | Patient Profile Agent, Notification Agent. | P1 | Proposed |
| WF-036 Fee Information Request | Provide approved administrative fee/payment-process information. | Billing inquiry. | Service, organization, locale, approved fee policy. | Grounded fee/process response or handoff. | Billing & Payment Agent, approved fee schedule. | P2 | Proposed |
| WF-037 Payment Initiation | Request an approved tokenized payment session after confirmation. | Confirmed payment request. | Authorized bill reference, amount, idempotency key, consent. | Payment session/reference or safe failure. | Billing & Payment Agent, future payment gateway. | P2 | Proposed |
| WF-038 Payment Reconciliation | Reconcile payment provider status with billing reference. | Provider callback or scheduled run. | Payment reference, provider status, idempotency data. | Settled/pending/exception state. | Future gateway, Billing Agent, audit store. | P0 | Proposed |
| WF-039 Payment Dispute Handoff | Route payment failure, refund, duplicate-charge, or dispute issues. | Patient/staff dispute or reconciliation exception. | Payment reference, safe summary, identity level. | Authorized billing task and acknowledgement. | Billing Agent, staff queue, WF-038. | P0 | Proposed |
| WF-040 Receipt Notification | Deliver an approved payment receipt or status notice. | Reconciled eligible payment state. | Payment reference, template, consent, channel preference. | Delivery result or exception. | WF-038, Notification Agent. | P2 | Proposed |

## Administration, Reporting, Security, and Operations

| ID & workflow | Purpose | Trigger | Inputs | Outputs | Dependencies | Priority | Status |
| --- | --- | --- | --- | --- | --- | --- | --- |
| WF-041 Staff Access Request | Capture and route staff access changes with separation of duties. | Authorized manager request. | Requester identity, role, scope, justification. | Approval task, grant/reject result, audit event. | Admin Agent, future RBAC, security owner. | P0 | Proposed |
| WF-042 Privileged Access Review | Periodically review elevated access and remediation actions. | Scheduled governance review. | Role assignments, last-use data, reviewer scope. | Review findings and action tasks. | Future RBAC, audit store, security queue. | P0 | Proposed |
| WF-043 Organization Configuration Change | Govern approved service, schedule, template, or policy setting changes. | Admin change request. | Authorized request, scope, version, approval. | Validated change task, audit event, rollback reference. | Admin Agent, configuration store, approval queue. | P1 | Proposed |
| WF-044 Workflow Failure Alert | Detect failed workflow execution and notify an operator. | Error, timeout, or retry exhaustion. | Workflow ID, correlation ID, safe error category, severity. | Alert, incident/task, retry decision. | n8n, monitoring, staff/operations queue. | P0 | Proposed |
| WF-045 Integration Health Check | Monitor approved provider availability and authentication health. | Scheduled health check. | Provider health endpoint/result, configuration scope. | Health state, alert, recovery task. | n8n, approved providers, monitoring. | P0 | Proposed |
| WF-046 Security Event Escalation | Route suspicious security events for containment and investigation. | Signature failure, unusual access, secret exposure signal, or policy alert. | Safe event metadata, correlation, severity. | Security incident task and alert. | Monitoring, audit store, security owner. | P0 | Proposed |
| WF-047 Audit Event Capture | Record a safe, structured audit event for material actions. | Approved workflow action/state change. | Actor type, action, timestamp, correlation, outcome. | Protected audit record. | n8n, future audit store. | P0 | Proposed |
| WF-048 Operational Report Generation | Produce authorized operational reports from approved aggregates. | Authorized on-demand/scheduled report request. | Report definition, role, organization, time range. | Report result/export task or denial. | Reporting Agent, pilot sheet/future analytics store. | P2 | Proposed |
| WF-049 Analytics Anomaly Detection | Detect unusual operational patterns for human review. | Scheduled analytics evaluation. | Approved aggregated metrics, baseline, data-quality status. | Anomaly insight and investigation task. | Analytics Agent, future analytics/monitoring store. | P2 | Proposed |
| WF-050 Backup Verification and Restore Test | Verify backup completion and record controlled restore-test evidence. | Scheduled backup check or approved drill. | Backup metadata, test environment, RPO/RTO targets. | Verification result, alert, remediation task. | Future backup service, monitoring, operations owner. | P0 | Proposed |

## Implementation Sequencing

The catalog does not authorize implementation. Subject to explicit approval, the expected pilot sequence is WF-005, WF-006, WF-007, WF-011 through WF-018, WF-023 through WF-026, and WF-047. Higher-numbered workflows may be selected earlier only with a documented dependency, security, and data-flow review.
