# EKMA AI Agent Architecture

**Status:** Design specification only.  
**Implementation gate:** No agent, prompt, code, n8n workflow, integration, or credential may be implemented or activated without explicit project-owner approval.

This document defines the logical AI-agent architecture for EKMA AI. It supplements and must comply with [MASTER_SPEC.md](MASTER_SPEC.md), which remains the project’s governing specification.

## Operating Model

Agents are bounded decision-support responsibilities, not independent authorities. They may classify, guide, retrieve approved administrative information, prepare actions, and request approved workflow execution. They may not diagnose, prescribe, triage, interpret medical results, determine emergencies, or replace staff judgment.

Every agent must operate with:

- a defined owner, purpose, allowed tools, data classification, and least-privilege permissions;
- a correlation ID, organization scope, channel context, and safe audit event for each interaction;
- minimum necessary data and redacted logging;
- explicit confirmation before consequential actions;
- a deterministic human-handoff route for uncertainty, sensitive content, or failure;
- versioned prompts, evaluation evidence, and rollback capability before release.

## Agent Communication and n8n

n8n is the initial orchestration boundary, not an unrestricted agent tool. An approved n8n workflow receives normalized events from an approved channel, validates them, assigns a correlation ID, and invokes only the agent capability appropriate to the request. The agent returns a structured decision—such as `respond`, `request_confirmation`, `handoff`, `create_action_request`, or `failure`—with minimum required context. n8n then enforces policy and performs approved provider actions.

```text
Channel event / staff request
        │
        ▼
Approved n8n workflow: authenticate → validate → normalize → correlate
        │
        ▼
Relevant agent(s): classify / retrieve / prepare decision
        │
        ▼
n8n policy gate: authorize → confirm → execute provider action or hand off
        │
        ▼
Provider, staff queue, or safe user response
        │
        ▼
Redacted audit event, metrics, and reconciliation state
```

Agents do not directly hold provider credentials, directly invoke Google Calendar, send a message, charge a payment method, or alter permissions. n8n (or a future approved service boundary) performs those actions using environment-scoped credentials after validation and authorization. Workflows must meet the n8n standards in `MASTER_SPEC.md`: idempotency, bounded retries, time-zone handling, signature verification, safe logging, human recovery, and version control without secrets.

## Shared Contract

### Minimum inputs

Each agent receives only what its task needs: organization ID, correlation ID, authenticated or asserted identity level, channel, locale, time zone, normalized request, consent/preferences where applicable, and approved relevant context.

### Standard outputs

Each agent returns a structured outcome containing: decision type, safe response or staff summary, requested next action, confidence/uncertainty indicator, required confirmation, handoff reason if applicable, and safe audit metadata. It must not return hidden reasoning, secrets, or unfiltered sensitive content.

### Shared handoff rules

All agents must hand off to an authorized human queue when a request is clinical, urgent, potentially emergent, legally sensitive, abusive or unsafe, identity-ambiguous, outside approved policy, high-impact, or cannot be completed reliably. The user must receive a safe administrative response that does not provide medical advice or delay access to appropriate emergency services.

### Shared failure handling

On invalid input, unavailable dependency, low confidence, policy denial, timeout, or duplicate event: do not guess; preserve the correlation ID; return a safe failure category; retry only where the workflow policy allows; create a staff task when needed; and record redacted telemetry. Consequential actions require reconciliation before any retry.

## 1. Reception Agent

| Aspect | Definition |
| --- | --- |
| Purpose | Be the first administrative conversational interface and route each request safely. |
| Responsibilities | Detect intent, language, organization context, and consent needs; provide approved administrative information; collect minimum required details; select the next approved agent or handoff. |
| Inputs | Normalized channel message, organization context, locale, asserted identity level, approved conversation context, and consent status. |
| Outputs | Safe response, normalized administrative intent, agent-routing request, confirmation request, or staff-handoff package. |
| Connected systems | n8n orchestration; approved channel adapters; future patient/profile, knowledge, and staff-queue services. |
| Permissions | Read approved organization policies and minimum conversation context; cannot book, send, charge, or modify records directly. |
| Human handoff rules | Hand off clinical/urgent requests, identity disputes, complaints, unsupported languages, policy uncertainty, repeated misunderstanding, and explicit staff requests. |
| Failure handling | Give a concise safe fallback, avoid inventing an answer, and create a staff task when the user needs resolution. |
| KPIs | Intent-routing accuracy, safe-handoff precision, first-response time, containment rate for approved requests, and complaint rate. |
| Future expansion | Multilingual support, authenticated web chat, organization-specific policy packs, and accessibility adaptations. |

## 2. Appointment Agent

| Aspect | Definition |
| --- | --- |
| Purpose | Guide booking, rescheduling, cancellation, and appointment-status requests under organizational policy. |
| Responsibilities | Gather required scheduling fields, explain approved options, request confirmation, prepare appointment action requests, and coordinate with Calendar Agent. |
| Inputs | Appointment intent, organization/service rules, requested date/time, location, identity level, and approved patient/contact reference. |
| Outputs | Availability inquiry, validated appointment action request, confirmation prompt, or staff handoff. |
| Connected systems | n8n; Calendar Agent; future PostgreSQL scheduling domain; approved staff queue. |
| Permissions | Read approved schedule/service policy and prepare actions; cannot independently create, alter, or cancel appointments. |
| Human handoff rules | Conflicts, double-booking risk, identity uncertainty, special accommodations, policy exceptions, provider outage, or clinical content. |
| Failure handling | Preserve the requested time and context safely, avoid promising a booking, and issue reconciliation/staff tasks after provider errors. |
| KPIs | Confirmed booking completion rate, duplicate-prevention rate, reschedule completion rate, calendar discrepancy rate, and staff-resolution time. |
| Future expansion | Waitlists, multi-resource booking, pre-visit administrative forms, and rules-based eligibility checks. |

## 3. Calendar Agent

| Aspect | Definition |
| --- | --- |
| Purpose | Translate approved scheduling requests into provider-neutral calendar operations and reconciliation decisions. |
| Responsibilities | Query availability, validate time zones and slot rules, prepare calendar create/update/cancel operations, and identify synchronization differences. |
| Inputs | Authorized appointment action request, service/practitioner/location constraints, idempotency key, and time-zone context. |
| Outputs | Availability result, provider operation request, conflict result, reconciliation task, or safe failure result. |
| Connected systems | n8n; Google Calendar initially; future scheduling database and alternative calendar providers. |
| Permissions | Read calendar availability and prepare approved provider operations; no direct credential access and no policy override. |
| Human handoff rules | Ambiguous availability, conflicting external edits, unsupported recurring/complex bookings, or a mismatch that affects a patient commitment. |
| Failure handling | Never infer success from a timeout; mark outcome unknown, reconcile provider state, and prevent duplicate execution with idempotency keys. |
| KPIs | Availability accuracy, operation success rate, reconciliation latency, duplicate event rate, and time-zone error rate. |
| Future expansion | Multiple calendars, practitioner capacity rules, resource rooms/equipment, and calendar-provider abstraction. |

## 4. Reminder Agent

| Aspect | Definition |
| --- | --- |
| Purpose | Determine whether an approved appointment reminder is due and prepare its content and schedule. |
| Responsibilities | Apply reminder policy, contact preferences, consent/opt-out rules, appointment status, locale, and timing constraints. |
| Inputs | Approved appointment state, notification preferences, reminder policy, channel availability, and time zone. |
| Outputs | Notification scheduling request, suppression decision, staff exception, or delivery-reconciliation request. |
| Connected systems | n8n; Appointment/Calendar Agent; Notification Agent; future preference store and PostgreSQL. |
| Permissions | Read minimum appointment and preference data; cannot send messages directly or alter appointments. |
| Human handoff rules | Sensitive reminder content, unclear consent, repeated delivery failure, patient opt-out dispute, or high-impact schedule change. |
| Failure handling | Suppress rather than send when consent or appointment state is uncertain; create a safe exception record. |
| KPIs | Timely reminder rate, suppression accuracy, opt-out compliance, delivery success rate, and no-show reduction where measurable. |
| Future expansion | Preference learning constrained by consent, adaptive timing, multi-step reminder campaigns, and staff-call tasks. |

## 5. Medical FAQ Agent

| Aspect | Definition |
| --- | --- |
| Purpose | Answer approved, non-clinical medical-center FAQs using governed organizational knowledge. |
| Responsibilities | Retrieve source-approved content; answer questions about services, hours, locations, preparation instructions that are explicitly approved, and administrative processes; cite or identify source currency where supported. |
| Inputs | Normalized question, organization scope, locale, approved knowledge corpus, and source metadata. |
| Outputs | Grounded administrative answer, clarification request, content-gap task, or staff handoff. |
| Connected systems | n8n; future RAG/knowledge service; organization content repository; staff queue. |
| Permissions | Read only approved, organization-scoped knowledge; no access to patient records, diagnostic systems, or external web sources by default. |
| Human handoff rules | Any diagnosis, symptom, test-result interpretation, treatment, urgency, contraindication, missing/conflicting/stale source, or clinical wording ambiguity. |
| Failure handling | State that it cannot confirm the answer and offer staff contact; never fabricate content or general medical advice. |
| KPIs | Grounded-answer rate, source freshness, unsupported-answer rate, clinical-escalation recall, and content-gap closure time. |
| Future expansion | Versioned Persian-first RAG, document approval workflows, source citations, and role-aware staff knowledge access. |

## 6. Patient Profile Agent

| Aspect | Definition |
| --- | --- |
| Purpose | Manage minimum necessary contact, preference, consent, and identity-context updates under strict controls. |
| Responsibilities | Request approved profile fields, validate format, prepare create/update requests, identify duplicates, and explain privacy choices. |
| Inputs | Authenticated/asserted identity level, requested profile change, consent state, organization policy, and existing minimal profile reference. |
| Outputs | Validated update request, identity-verification request, duplicate-review task, consent record request, or handoff. |
| Connected systems | n8n; Google Sheets only for approved pilot fields; future PostgreSQL identity/consent domain; staff queue. |
| Permissions | Read/update only approved, minimum fields through authorized workflows; cannot expose records, merge identities, or modify clinical information. |
| Human handoff rules | Identity mismatch, duplicate profile, representative access, deletion/access request, sensitive-data request, or any clinical record request. |
| Failure handling | Do not overwrite uncertain records; retain pending state, request verification, and route to authorized staff. |
| KPIs | Profile-update completion rate, duplicate detection precision, consent capture accuracy, verification completion time, and privacy-request SLA. |
| Future expansion | Verified identity providers, patient portal linkage, consent versioning, and data-subject request orchestration. |

## 7. Reporting Agent

| Aspect | Definition |
| --- | --- |
| Purpose | Produce approved operational reports and staff-ready summaries from authorized aggregated data. |
| Responsibilities | Interpret approved reporting requests, apply role and organization scope, aggregate data, redact/minimize sensitive values, and explain report limitations. |
| Inputs | Authorized user request, report definition, date range, organization scope, and role permissions. |
| Outputs | Approved report request/result, export request, summary, or denial/handoff explanation. |
| Connected systems | n8n; Google Sheets during pilot; future PostgreSQL analytics views, dashboard service, and audit store. |
| Permissions | Read only approved aggregated/reporting data for the requester’s scope; no unrestricted row-level patient data access. |
| Human handoff rules | Requests for sensitive/raw data, cross-organization comparison, ambiguous authorization, unusual export volume, or data-quality concern. |
| Failure handling | Mark reports incomplete/stale when sources fail; do not substitute estimates for authoritative metrics. |
| KPIs | Report correctness, generation latency, authorization-denial accuracy, data-freshness compliance, and self-service completion rate. |
| Future expansion | Scheduled reports, governed exports, role-specific dashboards, and natural-language reporting with semantic-layer controls. |

## 8. Billing & Payment Agent

| Aspect | Definition |
| --- | --- |
| Purpose | Explain approved billing processes and coordinate payment initiation and reconciliation without handling unnecessary payment secrets. |
| Responsibilities | Present approved fees/policies, create payment requests after confirmation, track reference status, and route disputes. |
| Inputs | Authorized billing context, approved fee schedule, payment intent, confirmation, and provider status/reference. |
| Outputs | Fee information, payment-initiation request, status/reconciliation request, receipt-routing request, or handoff. |
| Connected systems | n8n; future approved payment gateway, billing domain, notification service, and staff queue. |
| Permissions | Read approved fee/payment status and request tokenized payment operations; cannot access card data, settle/refund funds, or alter fees. |
| Human handoff rules | Disputes, refunds, chargebacks, failed/duplicate payments, identity uncertainty, exceptions, or any regulated financial issue. |
| Failure handling | Never claim payment success until provider-confirmed and reconciled; use idempotency and staff resolution for unknown states. |
| KPIs | Payment-initiation success, reconciliation accuracy, duplicate-charge prevention, dispute routing time, and payment-status accuracy. |
| Future expansion | Multiple gateways, installment policies, invoices, receipts, refunds through authorized staff approval, and accounting reconciliation. |

## 9. Notification Agent

| Aspect | Definition |
| --- | --- |
| Purpose | Deliver approved transactional communications through permitted channels and record delivery status. |
| Responsibilities | Select approved channel/template, enforce consent and opt-out, render minimum necessary content, send via n8n provider adapter, and track outcomes. |
| Inputs | Authorized notification request, recipient reference, channel preference, consent status, approved template, locale, and idempotency key. |
| Outputs | Send request, suppression result, delivery outcome, retry request, or staff exception. |
| Connected systems | n8n; future Telegram, Gmail, WhatsApp, Bale, SMS, voice, and web-chat adapters; preference store. |
| Permissions | Send only pre-authorized templated notifications via authorized workflow; no free-form sensitive messaging or provider credential access. |
| Human handoff rules | Consent ambiguity, sensitive/unapproved content, delivery failures beyond retry policy, recipient complaint, or channel-policy violation. |
| Failure handling | Use bounded retries; prevent duplicate delivery with idempotency; record delivery state and route unresolved messages to staff. |
| KPIs | Delivery rate, duplicate-send rate, opt-out compliance, time-to-delivery, bounce/failure rate, and complaint rate. |
| Future expansion | Channel preference optimization, template governance, multi-language templates, and fallback channels subject to consent. |

## 10. Voice Agent

| Aspect | Definition |
| --- | --- |
| Purpose | Provide approved voice-based administrative reception and route calls to staff when automation is unsuitable. |
| Responsibilities | Obtain required notices/consent, recognize administrative intent, gather minimal details, speak approved responses, and initiate staff transfer requests. |
| Inputs | Voice transcript or structured call event, caller context, language, consent status, organization policy, and approved conversation state. |
| Outputs | Spoken response content, normalized intent, callback/transfer request, appointment action request, or handoff. |
| Connected systems | n8n; future telephony provider, speech services, scheduling/notification agents, and staff call queue. |
| Permissions | Access only approved call context and administrative tools through n8n; cannot provide clinical advice, record calls without approved policy, or transfer without routing controls. |
| Human handoff rules | Emergency/clinical content, failed recognition, caller request, vulnerable-user need, prolonged loop, identity uncertainty, or sensitive payment discussion. |
| Failure handling | Clearly state limitations, offer staff transfer/callback path, preserve only approved minimum context, and avoid repeated automated loops. |
| KPIs | Successful intent capture, transfer completion, recognition failure rate, call containment for safe requests, average handling time, and user satisfaction. |
| Future expansion | Persian speech optimization, IVR integration, accessibility features, scheduled callbacks, and staff-assist transcription under policy. |

## 11. Admin Agent

| Aspect | Definition |
| --- | --- |
| Purpose | Assist authorized administrators with governed configuration and operational tasks without granting autonomous control. |
| Responsibilities | Explain settings, validate change requests, prepare approval records, summarize audit information, and route privileged actions through separation-of-duties controls. |
| Inputs | Authenticated administrator identity, role, organization scope, configuration request, policy version, and approval state. |
| Outputs | Validated change request, policy explanation, audit summary, approval task, or access denial/handoff. |
| Connected systems | n8n; future configuration service, identity/RBAC, audit store, monitoring, and staff/admin queue. |
| Permissions | Read approved scoped configuration and prepare changes; cannot directly change access roles, secrets, production settings, workflows, or retention policies. |
| Human handoff rules | Privilege escalation, security policy changes, secret handling, production-impacting change, ambiguous authority, or cross-organization request. |
| Failure handling | Deny by default, preserve audit evidence, and route authorized requests to the designated administrator/security owner. |
| KPIs | Correct authorization decisions, configuration-request completion time, unauthorized-change prevention, audit completeness, and rollback readiness. |
| Future expansion | Approval workflows, policy-as-data interface, access-review assistance, and configuration drift detection. |

## 12. Analytics Agent

| Aspect | Definition |
| --- | --- |
| Purpose | Generate privacy-preserving operational insights and detect trends requiring human review. |
| Responsibilities | Analyze approved aggregates, identify workflow bottlenecks and anomalies, propose investigations, and explain data limitations. |
| Inputs | Authorized aggregate metrics, time ranges, organization scope, data-quality metadata, and approved analytic definitions. |
| Outputs | Aggregated insight, anomaly alert, investigation request, dashboard narrative, or insufficient-data result. |
| Connected systems | n8n; future PostgreSQL analytics layer, monitoring platform, reporting service, and staff queue. |
| Permissions | Read approved de-identified or aggregated data only; no raw patient conversations or unrestricted cross-organization datasets. |
| Human handoff rules | Potential safety/security anomaly, unexpected data exposure, poor data quality affecting decisions, or recommendation with material operational impact. |
| Failure handling | Label uncertainty and incomplete data; do not create operational changes automatically; create review tasks for significant anomalies. |
| KPIs | Insight precision, anomaly false-positive rate, time-to-detection, data-freshness compliance, and investigation closure rate. |
| Future expansion | Forecasting with governance, capacity planning, privacy-preserving benchmarking, and controlled experimentation analysis. |

## Agent-to-Agent Coordination

Agents communicate only through approved, structured n8n-mediated requests or future versioned service contracts. The Reception Agent is the default entry router. The Appointment Agent owns the user journey for appointment intent; Calendar Agent handles calendar semantics; Reminder and Notification Agents own outbound messaging decisions and delivery respectively. Medical FAQ Agent is isolated from patient data. Reporting and Analytics Agents use governed data products rather than live operational stores. Admin Agent is never a bypass for privileged controls.

An agent may request another agent’s capability but cannot silently expand its own permissions. All cross-agent requests must retain organization scope, correlation ID, authorization context, data classification, and explicit purpose. Cyclic routing must be detected and capped; repeated uncertainty results in human handoff.

## Approval Before Implementation

This architecture is a design baseline only. Before implementing any agent or workflow, the project owner must explicitly approve the specific scope. The approved work must then define the agent prompt/version, permitted tools, data flow, retention, security review, workflow design, evaluation suite using synthetic data, KPIs, monitoring, runbook, rollback plan, and named operational owner.
