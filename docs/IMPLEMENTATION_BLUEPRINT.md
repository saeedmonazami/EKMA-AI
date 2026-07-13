# EKMA AI Implementation Blueprint

**Status:** Master implementation guide; documentation only.  
**Authority:** Implements the direction in `MASTER_SPEC.md`; no section authorizes code, n8n JSON, credentials, integrations, or production activation without an explicit owner approval.

## 1. Overall Implementation Strategy

Build EKMA AI incrementally around safe administrative reception, not clinical automation. Establish one thin, auditable vertical slice first—authenticated intake, patient matching, appointment availability, confirmed booking, Google Calendar reconciliation, Google Sheets pilot record, and human handoff—then harden operations before expanding channels or intelligence.

The implementation progression is:

1. Foundations: environments, secrets, access control, audit conventions, test data, operational ownership.
2. Controlled n8n/Google pilot: WF-001 and the minimum prerequisite operational controls.
3. Scheduling and notification reliability: reconciliation, cancellation/reschedule, reminders, handoffs.
4. PostgreSQL migration: canonical operational data, audit, permissions, and integrations.
5. Multi-channel and governed AI: normalized channel boundary, approved knowledge retrieval, staff operations.
6. Payments, analytics, scale, and resilience: monitored production platform with tested recovery.

No workflow can be activated before a named owner approves its data flow, credential scope, test evidence, alert route, manual recovery process, and rollback plan. Patient-facing features remain administrative only and route medical, urgent, ambiguous, or identity-sensitive content to staff.

## 2. Cross-Workflow Operating Profiles

The profile codes below are assigned to every workflow in the catalog.

| Profile | Definition |
| --- | --- |
| **R0** | No automatic retry for validation/policy failure; create a safe result or handoff. |
| **R1** | Read-only external operation: retry transient `429/5xx/network` failures up to two times with exponential backoff and jitter. |
| **R2** | Consequential external write: idempotency key, pre-action lookup, two bounded transient retries; unknown outcome requires reconciliation before any retry. |
| **R3** | Financial/security/retention write: R2 plus dual-control/manual review; never auto-compensate unless explicitly governed. |
| **A1** | Record start and terminal structured audit event: workflow/execution/correlation IDs, actor/source, organization, safe outcome/error, timestamp, retry count. |
| **A2** | A1 plus append-only entity history and protected access audit. |
| **H0** | No standard handoff; return safe operational result. |
| **H1** | Handoff for validation, provider failure beyond retry, policy uncertainty, or user request. |
| **H2** | H1 plus mandatory handoff for identity, privacy, clinical/urgent, financial, security, or cross-tenant concern. |

Every audit timestamp is `Asia/Tehran` with offset for operational use. Logs and audit events never contain credentials, raw National ID, raw PHI, full messages, or payment secrets.

## 3. Complete Implementation Order

| Sequence | Deliverable/workflow group | Exit gate |
| --- | --- | --- |
| 0 | Environment, secret, access, logging, audit, test-data, and operational runbook foundations | Security/owner approval; synthetic-data test environment ready. |
| 1 | WF-047 audit capture, WF-044 failure alert, WF-045 integration health, WF-007 handoff | Audit and manual recovery proven before patient actions. |
| 2 | WF-001 combined registration/booking pilot | Calendar/Sheets rollback, duplicate prevention, and Tehran-time tests pass. |
| 3 | WF-011–018 scheduling decomposition and reconciliation; WF-008 handoff resolution | Booking lifecycle and recovery are reliable. |
| 4 | WF-003–004, 026, 033–035 identity/consent/preferences | Consent, identity, and privacy controls reviewed. |
| 5 | WF-023–025, 027 notification and reminder foundation | Delivery/retry/opt-out metrics and staff recovery work. |
| 6 | PostgreSQL platform migration and WF-041–043 administrative governance | Source-of-record migration reconciled and restored successfully. |
| 7 | WF-005–006, 021–022, 028–030 multi-channel and knowledge intake | Channel controls and safety evaluation accepted. |
| 8 | WF-019–020, 031–032, 036–040 enhanced scheduling, voice, and payments | Provider policies, consent, and reconciliation accepted. |
| 9 | WF-048–050 reporting, analytics, backup/restore, scale hardening | Monitoring, backup restore, security review, and release readiness accepted. |

## 4. Workflow Dependency Graph

```text
Platform controls: WF-047 ─┬─> all workflows
                           ├─> WF-044, WF-045, WF-046, WF-050
                           └─> WF-007 ─> WF-008

Identity/consent: WF-001 ─> WF-002 ─> WF-003 ─> WF-004 ─> WF-026
       └────────────────────────────────────────> WF-033 ─> WF-034 / WF-035

Scheduling: WF-001 ─> WF-011 ─> WF-012 ─> WF-013 ─> WF-023 ─> WF-024 ─> WF-025
                         ├─> WF-014 / WF-015 / WF-016
                         ├─> WF-017 ─> WF-018
                         └─> WF-019 / WF-020

Channels: WF-005 ─> WF-006 ─┬─> WF-021 ─> WF-022
                              ├─> WF-028 / WF-029 / WF-030 / WF-031 ─> WF-032
                              └─> scheduling and handoff workflows

Payments: WF-036 ─> WF-037 ─> WF-038 ─> WF-039 / WF-040
Governance: WF-041 ─> WF-042; WF-043 ─> approved configuration consumers
Insights: WF-048 / WF-049 depend on PostgreSQL, audit, and monitoring; WF-050 supports all production services.
```

## 5. Workflow Catalog Implementation Details

Abbreviations: **Sys** = external systems; **Cred** = required credential class; **E/R/A/H** = error/retry/audit/handoff profile. Complexity: S (small), M (medium), L (large), XL (cross-domain/high-risk).

| ID / name | Purpose | Trigger | Inputs → outputs | Sys / Cred | E / R / A / H | Complexity |
| --- | --- | --- | --- | --- | --- | --- |
| WF-001 Patient Registration & Booking | Register/match patient, offer and confirm booking. | Authenticated register/confirm request. | ID/mobile, service/date → offer or Calendar confirmation. | n8n, Calendar, Sheets; webhook, Google OAuth, match-key secret. | validation/conflict/rollback; R2; A2; H2 | XL |
| WF-002 Identity Verification | Establish sufficient identity confidence. | Protected action/mismatch. | Assertions, method → assurance level or case. | Identity provider, PostgreSQL; provider/RBAC creds. | verify/deny; R1; A2; H2 | L |
| WF-003 Consent Capture | Capture versioned consent evidence. | Registration/policy prompt. | Notice/version, response → consent record. | PostgreSQL, channel; DB/channel creds. | invalid/absent consent; R2; A2; H2 | M |
| WF-004 Consent Withdrawal | Apply opt-out/withdrawal promptly. | Authenticated request. | Contact, scope → withdrawn state/suppression. | PostgreSQL, notification store; DB creds. | policy conflict; R3; A2; H2 | M |
| WF-005 Channel Message Intake | Authenticate and normalize inbound event. | Provider webhook. | Signed payload → normalized event/reject. | Channel adapter, n8n; provider webhook secret. | signature/input failure; R1; A1; H1 | M |
| WF-006 Reception Intent Routing | Route safe administrative request. | Normalized event. | Context/message → route/response/handoff. | n8n, AI policy; model credential if approved. | uncertain/unsafe intent; R0; A1; H2 | M |
| WF-007 Human Staff Handoff | Create/assign exception case. | Agent/policy/staff escalation. | Safe summary, priority → case/status. | Staff queue, PostgreSQL; queue/DB creds. | queue failure; R2; A2; H2 | M |
| WF-008 Handoff Resolution | Resolve and optionally notify case outcome. | Staff action. | Case/resolution → closed case/notification request. | Staff queue, notification, DB; scoped creds. | authorization/delivery failure; R2; A2; H1 | M |
| WF-009 Data Subject Request Intake | Register privacy access/correction/deletion case. | Verified privacy request. | Identity, request scope → compliance case. | PostgreSQL, staff queue; DB/RBAC creds. | verification/policy issue; R3; A2; H2 | L |
| WF-010 Emergency/Clinical Escalation | Stop automation and route unsafe content. | Safety detection/explicit request. | Safe context → priority handoff/response. | n8n, staff queue; queue credential. | queue unavailable; R2; A2; H2 | M |
| WF-011 Availability Lookup | Return approved availability. | Scheduling inquiry. | Service/date/time zone → slots/no availability. | Calendar, scheduling DB; Google/DB creds. | provider failure; R1; A1; H1 | M |
| WF-012 Book Appointment | Commit confirmed appointment. | Confirmed slot choice. | Identity, slot, idempotency key → appointment state. | Calendar, PostgreSQL; Google/DB creds. | conflict/unknown outcome; R2; A2; H2 | L |
| WF-013 Appointment Confirmation | Notify confirmed booking. | Confirmed/reconciled appointment. | Appointment/template/preference → delivery status. | Notification provider, DB; provider creds. | delivery/suppression; R2; A2; H1 | M |
| WF-014 Reschedule Appointment | Change appointment safely. | Confirmed reschedule request. | Appointment/new slot → updated state. | Calendar, DB; Google/DB creds. | conflict/unknown update; R2; A2; H2 | L |
| WF-015 Cancel Appointment | Cancel under policy. | Confirmed cancel request. | Appointment/identity → cancelled state/notice. | Calendar, DB, notifications; scoped creds. | policy/unknown update; R2; A2; H2 | M |
| WF-016 Appointment Status Lookup | Provide permitted appointment status. | Patient/staff inquiry. | Identity/reference → safe status. | DB/Calendar; scoped read creds. | identity/provider failure; R1; A1; H2 | S |
| WF-017 Calendar Change Reconciliation | Reconcile provider/internal differences. | Schedule/provider event. | External/internal state → resolved exception. | Calendar, DB; Google/DB creds. | mismatch; R2; A2; H2 | L |
| WF-018 Booking Conflict Resolution | Resolve duplicate/conflict/unknown booking. | Booking exception. | Operation/provider state → resolution/case. | Calendar, DB, staff queue; scoped creds. | unreconciled state; R2; A2; H2 | L |
| WF-019 Waitlist Management | Manage eligible future-slot requests. | Waitlist/freed slot event. | Criteria/preference → waitlist/offer. | DB, Calendar, notifications; DB/Google/provider creds. | stale criteria/delivery issue; R2; A2; H1 | L |
| WF-020 No-show Follow-up | Create approved administrative follow-up. | Staff no-show status. | Appointment/policy/consent → notification/case. | DB, notifications; DB/provider creds. | consent/delivery issue; R2; A2; H1 | M |
| WF-021 Administrative FAQ Response | Answer approved non-clinical FAQ. | FAQ intent. | Question/scope → grounded answer/handoff. | RAG/content store; model/RAG credentials. | missing/stale/clinical source; R0; A1; H2 | L |
| WF-022 Knowledge Content Gap | Track missing/conflicting content. | FAQ gap signal. | Category/source state → review case. | Content system, staff queue; scoped creds. | case creation failure; R2; A2; H1 | S |
| WF-023 Reminder Scheduling | Schedule or suppress reminder. | Appointment state/schedule. | Appointment, consent, policy → queued/suppressed reminder. | DB, scheduler; DB creds. | consent/state ambiguity; R2; A2; H2 | M |
| WF-024 Reminder Delivery | Send due reminder. | Reminder due event. | Template/preference → provider delivery. | Channel provider, DB; provider creds. | delivery/opt-out; R2; A2; H1 | M |
| WF-025 Delivery Reconciliation | Resolve delivery receipts/retries. | Provider callback/schedule. | Receipt/provider reference → final delivery state. | Channel provider, DB; provider creds. | unknown receipt; R2; A2; H1 | M |
| WF-026 Notification Opt-out Enforcement | Suppress prohibited sends. | Outbound notification request. | Contact/channel/purpose → send/suppress. | Consent DB, notification service; DB creds. | consent unavailable; R0; A2; H2 | M |
| WF-027 Service Change Notification | Notify affected appointments. | Staff-approved service change. | Change/affected set → deliveries/results. | DB, notification providers; scoped creds. | scale/delivery failure; R2; A2; H1 | M |
| WF-028 Website Chat Session Intake | Normalize web chat session. | Webchat event. | Session/message/consent → normalized route. | Webchat adapter, n8n; webhook creds. | validation/signature failure; R1; A1; H1 | M |
| WF-029 Email Reception Intake | Normalize inbound email. | Gmail event. | Sender/message metadata → route/handoff. | Gmail, n8n; Gmail OAuth. | spoofing/attachment policy; R1; A1; H2 | M |
| WF-030 Messaging Channel Intake | Normalize Telegram/WhatsApp/Bale/SMS. | Provider webhook. | Signed message → normalized route. | Channel adapters; provider secrets/OAuth. | signature/duplicate; R1; A1; H1 | L |
| WF-031 Voice Call Intake | Normalize voice call for admin reception. | Telephony event. | Call/transcript/consent → response/route. | Telephony, speech services; provider creds. | recognition/emergency issue; R1; A1; H2 | XL |
| WF-032 Voice-to-Staff Transfer | Transfer/callback escalation. | Caller/agent escalation. | Safe summary/routing → transfer/callback task. | Telephony, staff queue; provider creds. | transfer failure; R2; A2; H2 | M |
| WF-033 Patient Profile Update | Change permitted profile/preferences. | Verified request. | Identity/change → updated/pending profile. | PostgreSQL, identity; DB creds. | mismatch/duplicate; R2; A2; H2 | M |
| WF-034 Duplicate Profile Review | Review possible duplicates. | Matching signal. | Protected match data → review case. | PostgreSQL, staff queue; DB creds. | uncertain match; R0; A2; H2 | M |
| WF-035 Communication Preference Update | Update channel/language choices. | Verified request. | Preferences/consent → updated state. | DB, notification policy; DB creds. | consent conflict; R2; A2; H2 | S |
| WF-036 Fee Information Request | Provide approved fees/process. | Billing inquiry. | Service/scope → approved fee response. | Fee store/RAG; DB/RAG creds. | missing/stale data; R1; A1; H1 | S |
| WF-037 Payment Initiation | Start tokenized payment session. | Confirmed payment intent. | Bill/amount/idempotency → provider session. | Payment gateway, DB; gateway creds. | provider/amount mismatch; R3; A2; H2 | L |
| WF-038 Payment Reconciliation | Reconcile payment status. | Gateway callback/schedule. | Provider reference/status → canonical state. | Gateway, DB; gateway creds. | unknown/duplicate event; R3; A2; H2 | L |
| WF-039 Payment Dispute Handoff | Create dispute/refund case. | Dispute/exception. | Payment reference/safe summary → billing case. | DB, staff queue; scoped creds. | identity/financial dispute; R3; A2; H2 | M |
| WF-040 Receipt Notification | Send eligible receipt/status. | Reconciled payment. | Payment state/template → delivery result. | Provider, DB; provider creds. | delivery/consent issue; R2; A2; H1 | S |
| WF-041 Staff Access Request | Govern staff access lifecycle. | Manager request. | Subject/role/justification → approval result. | RBAC, DB; admin creds. | authority mismatch; R3; A2; H2 | L |
| WF-042 Privileged Access Review | Review elevated access. | Scheduled review. | Assignments/usage → findings/tasks. | RBAC, audit DB; security creds. | incomplete evidence; R3; A2; H2 | M |
| WF-043 Organization Config Change | Govern operational configuration changes. | Authorized request. | Config/version/approval → change/rollback ref. | Config store, DB; admin creds. | approval/validation issue; R3; A2; H2 | L |
| WF-044 Workflow Failure Alert | Alert on failed execution. | n8n error/timeout. | Execution/safe error → alert/task. | n8n, monitoring, queue; alert creds. | alert delivery failure; R2; A2; H1 | M |
| WF-045 Integration Health Check | Track provider health/auth. | Scheduled check. | Provider health result → status/alert. | Providers, monitoring; provider creds. | health failure; R1; A1; H1 | M |
| WF-046 Security Event Escalation | Create security incident. | Security signal. | Safe event/severity → incident/alert. | Monitoring, audit, security queue; security creds. | delivery failure; R3; A2; H2 | M |
| WF-047 Audit Event Capture | Append material audit evidence. | Any material state/action. | Actor/action/outcome → immutable audit event. | Audit DB/Sheets pilot; DB creds. | audit store unavailable: fail closed for writes. R2; A2; H2 | L |
| WF-048 Operational Report Generation | Produce authorized aggregate report. | Authorized request/schedule. | Scope/range/report → report/export. | Analytics views, DB; reporting creds. | stale/unauthorized data; R1; A2; H2 | M |
| WF-049 Analytics Anomaly Detection | Detect operational anomalies. | Scheduled analysis. | Aggregates/baseline → insight/case. | Analytics, monitoring; read creds. | data-quality issue; R1; A2; H1 | M |
| WF-050 Backup Verification & Restore Test | Verify backups/restores. | Schedule/approved drill. | Backup metadata/RPO-RTO → evidence/alert. | Backup store, monitoring; backup creds. | failed verification; R3; A2; H2 | L |

## 6. AI Agent to Workflow Mapping

| Agent | Primary workflows | Rules |
| --- | --- | --- |
| Reception Agent | WF-005, 006, 010, 028–031 | Routes only; never directly books, sends, or changes data. |
| Appointment Agent | WF-001, 011–020 | Requires confirmation and Calendar Agent/provider reconciliation. |
| Calendar Agent | WF-001, 011–018 | Owns calendar semantics, time zone, idempotency, and reconciliation decisions. |
| Reminder Agent | WF-023–025 | Applies consent and appointment-state rules before any send. |
| Medical FAQ Agent | WF-021–022, 036 | Uses approved non-clinical knowledge only; clinical content always hands off. |
| Patient Profile Agent | WF-001–004, 009, 033–035 | Controls minimum data, identity uncertainty, preferences, and consent. |
| Reporting Agent | WF-048 | Reads approved aggregate data only. |
| Billing & Payment Agent | WF-036–040 | Never accesses payment credentials or settles/refunds autonomously. |
| Notification Agent | WF-013, 023–027, 040 | Enforces templates, consent, opt-out, and delivery reconciliation. |
| Voice Agent | WF-031–032 | Must transfer on emergency, clinical, recognition, or caller-request conditions. |
| Admin Agent | WF-041–043 | Cannot bypass approval, secrets, or separation of duties. |
| Analytics Agent | WF-049–050 | Produces reviewable aggregate insights; makes no automatic operational change. |

## 7. Google Calendar Integration Strategy

Use Google Calendar as the initial scheduling provider, not the long-term patient database. Configure one dedicated Google identity per environment; share only allowlisted doctor calendars. Maintain a provider map from internal doctor/service key to Calendar ID, work hours, buffers, duration rules, location, and time zone.

All consequential Calendar operations use: an operation key, a deterministic provider event ID where supported, pre-read of the selected interval, immediate post-write verification, and reconciliation of ambiguous outcomes. Events use `Asia/Tehran`, minimum administrative content, and no raw identifiers beyond the clinic-approved minimum. Calendar changes are reconciled by WF-017; cancellation/reschedule use state/history and must not assume success after a timeout.

## 8. Google Sheets Strategy

Google Sheets is permitted only for the controlled pilot. Use a dedicated spreadsheet with restricted sharing, four approved worksheets for WF-001, protected headers, no secrets, no raw National ID, no public links, and documented retention. Treat it as a temporary operational register.

Every write is keyed, lookup-before-write, and audit-backed. Because Sheets has no row-level locking, serialize booking critical sections and migrate to PostgreSQL before multi-channel scale. Limit Sheets reporting to approved aggregates. Conduct a export/reconciliation and a restore test before each migration phase.

## 9. PostgreSQL Migration Strategy

1. Approve the physical data dictionary, RLS policy, retention policy, encryption/key management, backups, and migrations derived from `DATABASE_SCHEMA.md`.
2. Create isolated development/staging databases and migration tooling; test only synthetic data.
3. Implement canonical domains in order: organizations/RBAC → contacts/consent → services/schedules → appointments/history → conversations/handoffs → notifications → audit/integrations → payments/knowledge.
4. Add dual-read reconciliation from Sheets/Calendar to PostgreSQL; do not dual-write production booking state without an idempotency/reconciliation plan.
5. Run historical pilot-data cleansing, protected transform, validation counts, duplicate review, and signed migration acceptance.
6. Cut over a single organization with read-only Sheets fallback, provider reconciliation, monitoring, and rollback window.
7. Decommission Sheets as system of record only after data validation, backup, retention disposition, and owner approval.

## 10. Notification Strategy

Create a provider-neutral notification contract: notification ID, organization/contact reference, purpose, approved template version, locale, consent state, channel, idempotency key, scheduled time, delivery attempts, and terminal status. WF-026 is a mandatory policy gate before every send. Providers are adapters; template policy and consent logic are centralized.

Start with confirmation response through the initiating pilot channel. Add external messaging only after provider approval, opt-out behavior, delivery reconciliation, abuse/rate limits, and a human recovery path are proven. Never use notifications for clinical advice or unreviewed sensitive content.

## 11. Authentication Strategy

Authenticate every inbound webhook with a provider signature or dedicated secret/gateway identity, enforce replay protection and rate limits, then attach organization and channel context server-side. Human staff authenticate through an approved identity provider with MFA for privileged roles, role-based access, short-lived sessions, and audit events.

Patient identity assurance is proportional to action risk: low-risk public information requires no profile lookup; appointment changes, profile changes, privacy requests, and payments require the approved verification level. Never trust caller-supplied organization ID, role, or provider identifiers without server-side validation.

## 12. Multi-Channel Strategy

| Channel | Integration pattern | Preconditions |
| --- | --- | --- |
| Telegram | Signed webhook adapter → WF-005 → WF-006. | Bot policy, consent, rate limits, delivery rules. |
| Website chat | Authenticated session/webhook adapter → WF-028. | Consent notice, session security, abuse protection. |
| Gmail | Scoped inbound mailbox adapter → WF-029. | Sender/attachment policy, phishing controls, retention. |
| WhatsApp | Official approved provider adapter → WF-030. | Template approval, opt-in/opt-out, provider review. |
| Bale | Approved provider adapter → WF-030. | Authentication, privacy, consent, delivery review. |
| SMS | Provider adapter → WF-030/024. | Consent, template, opt-out, cost/rate controls. |
| Voice | Telephony + speech adapter → WF-031/032. | Recording/consent policy, transfer/callback route, emergency handoff. |

All channels normalize to one inbound event contract and one outbound notification contract. Channel adapters own transport authentication; core policy owns consent, safety, routing, and audit. Launch one channel at a time with sandbox tests and a rollback switch.

## 13. Logging Architecture

Use structured, redacted logs emitted by n8n and future services with: timestamp, environment, workflow/service, version, execution/correlation ID, organization scope, channel, action, status, duration, retry count, and safe error category. Maintain a separate append-oriented audit store for material actions; operational logs are not a substitute.

Use a centralized log collector in production, role-restricted access, retention policy, alert rules, protected incident preservation, and correlation across n8n, integrations, databases, and monitoring. Prohibit secrets, raw identifiers, full messages, raw PHI, and payment data from both logs and alert payloads.

## 14. Monitoring Architecture

Monitor n8n health, queue depth, execution duration, failure/retry rate, webhook rejection rate, Calendar/Sheets/provider health, booking conflicts, duplicate prevention, handoff backlog/SLA, notification delivery, database performance, backup success, credential expiry, and security events.

Use health checks, metrics collection, dashboards, alert routing with ownership and runbooks, and severity tiers. Alerts must be actionable and rate-limited. Production readiness requires synthetic end-to-end probes for booking and provider connectivity that do not create live patient data.

## 15. Backup Strategy

Back up n8n persistent data, approved workflow exports/configuration, PostgreSQL, audit data, and approved operational configuration. Encrypt backups, separate them from primary infrastructure, restrict access, define RPO/RTO, and verify completion automatically through WF-050.

Test restore in a non-production environment at a documented interval. A restore drill includes n8n configuration, PostgreSQL consistency, secret re-injection, Calendar/provider reconciliation, and evidence capture. Google Sheets pilot exports are protected and retained under the approved policy; they do not replace database backups.

## 16. Production Deployment Roadmap

1. Local development with synthetic data and no production credentials.
2. Staging Docker environment with isolated Google test resources, webhook authentication, logging, and manual recovery testing.
3. Single-clinic pilot: limited users, Calendar/Sheets, named staff owner, daily reconciliation, and feature flag/disable path.
4. Hardened staging with PostgreSQL, backups, monitoring, reverse proxy, secret management, and restore test.
5. Controlled production cutover to PostgreSQL with one organization and audited operations.
6. Add channels, RAG, payments, and multi-organization capacity one approved integration at a time.

Every stage requires a rollback plan, backup verification, synthetic test evidence, operational acceptance, and security review proportional to risk.

## 17. Future Microservice Separation

Keep n8n as orchestration while bounded domains mature. Extract only when ownership, scale, reliability, or security requires it:

- Identity/consent service
- Scheduling and calendar-adapter service
- Channel gateway and notification service
- Conversation/handoff service
- Knowledge/RAG service
- Payment and reconciliation service
- Audit/observability service
- Reporting/analytics service

Services communicate through versioned contracts, canonical IDs, idempotency keys, correlation IDs, and an event/outbox pattern once PostgreSQL is the system of record. Do not split services merely for organizational preference; retain simple boundaries until scaling evidence exists.

## 18. Recommended Project Milestones

| Milestone | Outcome |
| --- | --- |
| M0 Foundation accepted | Documentation, security/privacy baseline, owners, and environment strategy approved. |
| M1 Pilot safety controls | Audit, handoff, alerting, Calendar/Sheets test resources, and runbooks tested. |
| M2 WF-001 pilot | Registration/booking works with duplicate protection, reconciliation, and rollback. |
| M3 Scheduling lifecycle | Availability, booking, change, cancellation, reconciliation, reminders operational. |
| M4 Data foundation | PostgreSQL/RBAC/audit/backups and Sheets migration accepted. |
| M5 Multi-channel reception | One additional channel plus FAQ/handoff and notification policy accepted. |
| M6 Payments and intelligence | Governed RAG and payment integration accepted independently. |
| M7 Scale readiness | Monitoring, restore drills, security review, capacity/load evidence, multi-organization controls. |

## 19. Suggested Git Repository Structure

```text
EKMA-AI/
├── docs/                         # Governing specs, decisions, runbooks, workflow docs
│   ├── workflows/                 # One approved design and test plan per workflow
│   └── adr/                       # Future architecture decision records
├── n8n/                           # Approved sanitized workflow exports only
│   ├── workflows/
│   └── tests/
├── infrastructure/                # Approved Compose, deployment, monitoring, backup assets
│   ├── docker/
│   ├── monitoring/
│   └── backup/
├── database/                      # Future migrations, seeds (synthetic), data dictionary
│   ├── migrations/
│   └── tests/
├── services/                      # Future bounded application services
│   ├── scheduling/
│   ├── channels/
│   ├── identity/
│   └── notifications/
├── prompts/                       # Approved/versioned agent prompts and evaluations
├── test-data/                     # Synthetic fixtures only
└── scripts/                       # Approved operational utilities; no secrets
```

Until each implementation area is explicitly approved, its directory remains empty or contains only documentation. Never commit runtime data, credentials, real patient data, or unredacted provider exports.

## 20. Recommended Testing Strategy

Use a test pyramid adapted to integration workflows:

- Unit/logic tests for validation, time-zone arithmetic, idempotency key derivation, policy evaluation, and redaction.
- Contract tests for Calendar, Sheets, channel, payment, and identity adapters using sandbox accounts and synthetic fixtures.
- Workflow tests for every normal, validation, retry, duplicate, timeout, provider-conflict, handoff, audit, and rollback path.
- End-to-end synthetic journeys for registration, booking, reminder, cancellation, consent withdrawal, and staff resolution.
- Security tests for webhook signatures, authorization, replay, tenant isolation, secret leakage, prompt injection, and logging redaction.
- Load and resilience tests before multi-channel launch, including queue backlog, Calendar limits, provider outages, restore drills, and failover.
- AI evaluations for administrative scope, clinical-escalation recall, grounded FAQ answers, multilingual behavior, and adversarial prompts.

Test evidence is versioned with the approved workflow/service. Production data is never used unless a separately approved, legally compliant process permits it.

## 21. Recommended Implementation Sequence

Implement one production-quality vertical slice at a time:

1. Approve environment/credential/runbook design.
2. Implement and test WF-047, WF-044, WF-045, and WF-007.
3. Generate, review, and test WF-001 JSON only after the dedicated WF-001 approval gate.
4. Operate the pilot with daily reconciliation; fix observed reliability gaps before adding scope.
5. Implement appointment lifecycle and consent/preferences.
6. Move the operational record to PostgreSQL before broad channel expansion.
7. Add notifications, then one channel at a time.
8. Add governed FAQ/RAG, payments, voice, analytics, and multi-organization capabilities only after their prerequisite policies, observability, and recovery procedures are accepted.

This sequencing intentionally prioritizes safety, data integrity, and recovery over feature breadth.

## Approval Gate

This blueprint creates no code, JSON, n8n workflow, infrastructure asset, credential, integration, or database. Each implementation step remains subject to explicit project-owner approval.
