# WF-001 Technical Implementation Specification

**Status:** Pre-implementation blueprint.  
**Scope:** One n8n workflow; Google Calendar and Google Sheets only.  
**Not included:** n8n JSON, runnable configuration, credentials, Google resources, PostgreSQL, messaging providers, voice, or application code.

This document operationalizes [WF-001_PATIENT_REGISTRATION_AND_APPOINTMENT_BOOKING.md](WF-001_PATIENT_REGISTRATION_AND_APPOINTMENT_BOOKING.md). If there is a conflict, the WF-001 design document and `MASTER_SPEC.md` prevail.

## 1. Execution Order

The workflow has one authenticated webhook and two logical paths. Every path begins with a durable audit-start attempt and ends with exactly one terminal audit outcome (`COMPLETED`, `FAILED`, `HANDOFF`, or `ROLLBACK`) unless the audit store is unavailable before any booking action, in which case it fails closed.

```text
Webhook → context → audit STARTED → normalize/protect → authorize → action switch
  register: validate → idempotency lookup → patient match → upsert patient → availability
            → calculate slots → pending booking upsert → audit COMPLETED → respond offer
  confirm:  validate → pending lookup → state/idempotency checks → reservation marker
            → final availability check → create/retrieve Calendar event → Sheets confirmation
            → audit COMPLETED → respond confirmation
  any error: safe state update/handoff → audit terminal event → safe response
  post-create Sheets/audit failure: verify event → delete event → verify deletion → audit ROLLBACK/handoff
```

The confirmation critical section—from the booking-operation reservation through Calendar create/retrieve—must be serialized per `doctor_calendar_key` by n8n queue/concurrency configuration or an approved equivalent lock. Google Sheets alone cannot provide transaction locking.

## 2. Node Specification

“Error path” means the node connection to the shared error, handoff, or rollback branch. No node may use an unconnected `continue on fail` result.

| # | Node name | Node type | Purpose | Inputs | Outputs | Error path |
| --- | --- | --- | --- | --- | --- |
| 1 | Webhook — Receive Booking Request | Webhook | Receive one HTTPS POST request. | Request body, headers, request metadata. | `body`, `headers`, `received_at`. | Safe response → terminal audit where possible. |
| 2 | Set — Create Correlation Context | Set | Establish `correlation_id`, `execution_id`, `timezone=Asia/Tehran`, and Tehran timestamp. | Node 1 output. | `ctx` object merged with request. | Safe failure response; no provider call. |
| 3 | Google Sheets — Audit Log: Execution Started | Google Sheets | Idempotently append `STARTED` audit record. | `ctx`, action/request ID, audit operation key. | `audit_started=true`, audit row reference. | Fail closed → safe response. |
| 4 | Code — Normalize and Protect Input | Code | Validate/normalize National ID/mobile, derive protected match keys, normalize dates. | Request body, `MATCH_KEY_SECRET`, timezone. | `normalized`, `national_id_match_key`, `mobile_match_key`, validation result. | Manual handoff → terminal audit. |
| 5 | IF — Validate Webhook Authorization | IF | Enforce approved caller authentication. | Normalized headers/auth result. | Authorized or unauthorized branch. | Terminal audit `FAILED` → `401/403`. |
| 6 | Switch — Route Action | Switch | Permit only `register` or `confirm`. | `normalized.action`. | Register or confirm branch. | Manual handoff → terminal audit → `400`. |
| 7 | IF — Validate Registration Fields | IF | Require valid registration fields and reject unsupported clinical/free text. | Normalized register request. | Valid registration branch. | Manual handoff → terminal audit → safe response. |
| 8 | Google Sheets — Lookup Registration Request | Google Sheets | Find existing booking by `registration_request_id`. | Spreadsheet ID, `registration_request_id`. | Matching appointment row(s) or none. | Terminal audit `FAILED` → safe temporary response. |
| 9 | IF — Registration Request Reused | IF | Return prior offer if identical request already has valid result; reject ID/payload mismatch. | Node 8 row, request fingerprint. | Reuse, mismatch, or new-request branch. | Mismatch → handoff/audit; not a new booking. |
| 10 | Google Sheets — Lookup Existing Patient | Google Sheets | Look up `patients_pilot` by each protected match key. | Spreadsheet ID, two match keys. | National-ID match, mobile match, or no match. | Terminal audit `FAILED` → safe temporary response. |
| 11 | Code — Resolve Patient Identity | Code | Apply match decision rules and create patient operation key if new. | Node 10 matches, normalized data. | Existing `patient_pilot_id`, `new_patient`, or `IDENTITY_CONFLICT`. | Identity conflict → handoff → audit. |
| 12 | Google Sheets — Upsert Patient Record | Google Sheets | Idempotently create one new patient row or reuse existing row. | Patient operation key, safe patient fields, match keys. | `patient_pilot_id`, patient row reference. | Terminal audit `FAILED` → no Calendar call. |
| 13 | Set — Resolve Calendar Configuration | Set | Convert doctor key to allowlisted Calendar ID and slot rules. | `doctor_calendar_id` input key, environment configuration. | `calendar_id`, doctor label, work hours, duration, buffers, location. | Manual handoff → audit; do not query Calendar. |
| 14 | Google Calendar — Query Busy Events | Google Calendar | Read busy events for Tehran-local day. | Calendar ID, `timeMin`, `timeMax`. | Busy intervals. | Retry branch; then terminal audit `FAILED`. |
| 15 | Code — Calculate Eligible Slots | Code | Calculate eligible slots from busy intervals and rules. | Busy intervals, hours, buffers, date, duration. | Ordered slot array, expiry. | Terminal audit `FAILED`. |
| 16 | IF — Slots Available | IF | Require one or more available slots. | Slot array. | Offer branch or no-slots branch. | No-slots → terminal audit `COMPLETED` → safe response. |
| 17 | Set — Create Pending Booking Record | Set | Create opaque token, TTL, operation key, and `PENDING_CONFIRMATION` record. | Patient, slots, config, context. | Pending row payload. | Terminal audit `FAILED`. |
| 18 | Google Sheets — Upsert Pending Record | Google Sheets | Idempotently persist appointment offer. | Pending operation key, appointment payload. | Appointment row reference. | Terminal audit `FAILED`; no Calendar event exists. |
| 19 | Google Sheets — Audit Log: Slots Offered | Google Sheets | Idempotently append terminal `COMPLETED/SLOTS_OFFERED`. | Safe refs, execution context. | Audit row reference. | Safe failure response; no Calendar event exists. |
| 20 | Respond to Webhook — Offer Slots | Respond to Webhook | Return unreserved slots and token. | Slot array, token, expiry, Tehran timezone. | HTTP success response. | N/A. |
| 21 | IF — Validate Confirmation Fields | IF | Validate confirmation action, token, slot, request ID, affirmative confirmation. | Normalized confirm request. | Valid-confirmation branch. | Manual handoff → terminal audit → safe response. |
| 22 | Google Sheets — Lookup Pending Booking | Google Sheets | Retrieve appointment row by protected token lookup value. | Spreadsheet ID, booking token lookup. | One pending row or no row. | Unknown token → handoff/audit without disclosure; provider failure → terminal failure. |
| 23 | IF — Validate Pending State | IF | Check expiry, offered slot membership, status, and config consistency. | Pending row, selected slot, now Tehran. | Valid state or expired/conflict branch. | Update safe state/handoff → terminal audit. |
| 24 | Google Sheets — Lookup Confirmation Request | Google Sheets | Find prior confirmation by request/operation key. | Spreadsheet ID, confirmation request ID. | Prior result or none. | Terminal audit `FAILED` → safe response. |
| 25 | IF — Confirmation Request Reused | IF | Return prior confirmed/pending result; reject request-ID payload mismatch. | Node 24 result, request fingerprint. | Reuse, mismatch, or new confirmation. | Mismatch → handoff/audit. |
| 26 | Google Sheets — Reserve Booking Operation | Google Sheets | Idempotently mark this row `BOOKING_IN_PROGRESS` before Calendar call. | Booking row reference, calendar operation key, confirmation key. | Reserved row/state. | Terminal audit `FAILED`; no Calendar call. |
| 27 | Google Calendar — Re-check Busy Events | Google Calendar | Query exact selected interval immediately before create. | Calendar ID, selected start/end. | Busy/free result. | Retry branch; then terminal failure. |
| 28 | IF — Selected Slot Still Free | IF | Prevent Calendar create if interval is busy. | Node 27 events. | Create/retrieve branch or conflict branch. | Conflict Sheets update → audit terminal → safe response. |
| 29 | Google Calendar — Retrieve Event by Deterministic ID | Google Calendar or HTTP Request | Determine whether prior attempt already created event. | Calendar ID, deterministic event ID. | Existing event or not found. | Unknown provider failure → reconciliation handoff/audit; no create. |
| 30 | IF — Calendar Event Exists | IF | Reuse verified existing workflow-owned event or proceed to create. | Node 29 event and operation marker. | Existing event or create branch. | Ownership mismatch → urgent handoff/audit. |
| 31 | Google Calendar — Create Appointment Event | Google Calendar or HTTP Request | Create event with deterministic client event ID and safe metadata. | Calendar ID, deterministic event ID, start/end Tehran, safe title/location, operation marker. | Verified event reference. | Unknown outcome → node 29 reconciliation; never blind retry. |
| 32 | Google Sheets — Update Confirmed Booking | Google Sheets | Idempotently mark appointment `CONFIRMED`. | Appointment row, event reference, confirmation ID, Tehran timestamps. | Confirmed row. | Post-create rollback branch. |
| 33 | Google Sheets — Audit Log: Confirmed | Google Sheets | Append terminal `COMPLETED/CONFIRMED` audit entry. | Safe appointment/event refs, context. | Audit row reference. | Post-create rollback branch. |
| 34 | Respond to Webhook — Confirmation Message | Respond to Webhook | Return safe confirmed appointment information. | Doctor label, Tehran start/end, safe reference. | HTTP success response. | N/A. |
| 35 | Google Sheets — Create Manual Handoff | Google Sheets | Idempotently create staff case for validation, identity, or recovery exception. | Handoff ID, reason, priority, safe refs. | Handoff row reference. | Terminal audit failure; alert operator if store unavailable. |
| 36 | Google Sheets — Update Failure/Conflict State | Google Sheets | Persist safe failure/conflict/expired state. | Appointment row, status, safe error, timestamp. | Updated row reference. | Terminal audit/alert; never create a Calendar event. |
| 37 | Google Calendar — Roll Back Created Event | Google Calendar or HTTP Request | Delete only verified, workflow-owned event after post-create persistence failure. | Calendar ID, deterministic event ID, operation marker. | Delete verified or unknown/failure. | Urgent recovery handoff + terminal audit attempt. |
| 38 | Google Sheets — Audit Log: Terminal Event | Google Sheets | Append idempotent `FAILED`, `HANDOFF`, or `ROLLBACK` evidence. | Safe context, outcome, retry count. | Audit row reference. | If pre-create: fail closed; if post-create: rollback/handoff. |
| 39 | Error Trigger — WF-001 Operational Alert | Error Trigger | Invoke the approved error workflow with redacted execution metadata. | Execution ID, correlation ID, safe category. | Operator alert request. | Must not expose sensitive payload. |
| 40 | Respond to Webhook — Safe Error/Handoff | Respond to Webhook | Provide non-sensitive failure/handoff response. | Safe error code, correlation ID, local language message. | HTTP error/accepted response. | N/A. |

## 3. Google Calendar Operations

| Operation | Used by node(s) | Required behavior |
| --- | --- | --- |
| Read busy events for day | 14 | Query only allowlisted doctor calendar. Use Tehran-day start/end with offset. Exclude cancelled events and respect configured buffers in slot calculation. |
| Read busy events for selected interval | 27 | Query exactly the selected slot before create, within the serialized critical section. |
| Get event by deterministic ID | 29 | Retrieve event before any ambiguous-create retry. Verify its safe operation marker matches `calendar_operation_key`; otherwise stop and hand off. |
| Create event with client ID | 31 | Create non-recurring event with deterministic event ID, start/end in `Asia/Tehran`, safe title, configured location, and safe operation marker. No National ID, phone, raw payload, or clinical content. |
| Delete event for rollback | 37 | Only delete an event retrieved by deterministic ID and verified as owned by the current operation. Verify deletion after the call. |

The event-create/get/delete operations must use an n8n Google Calendar node only if the deployed version supports client-supplied event IDs and retrieval by ID. Otherwise use n8n’s HTTP Request node with the approved Google OAuth2 credential and Google Calendar API endpoints. This is a node selection constraint, not authorization to configure it.

## 4. Google Sheets Operations

All operations target one configured spreadsheet and an allowlisted worksheet. Lookup-before-write is mandatory, and each append/upsert uses its deterministic operation key.

| Operation | Worksheet | Nodes | Key / behavior |
| --- | --- | --- | --- |
| Audit start/terminal append | `audit_log` | 3, 19, 33, 38 | Lookup `audit_event_id`; append only if absent. Never update old audit events. |
| Registration lookup | `appointments_pilot` | 8 | Find by `registration_request_id`; compare request fingerprint before reuse. |
| Patient lookup | `patients_pilot` | 10 | Lookup separately by protected National-ID/mobile keys; never use raw identifier as a lookup column. |
| Patient upsert | `patients_pilot` | 12 | Re-read by `patient_operation_key` or match keys before append; append one active row only when no match exists. |
| Pending appointment upsert | `appointments_pilot` | 18 | Find by `pending_booking_operation_key` / registration request ID, then append/update `PENDING_CONFIRMATION`. |
| Pending appointment lookup | `appointments_pilot` | 22 | Find by protected booking-token lookup representation; return exactly one row. |
| Confirmation lookup/reservation | `appointments_pilot` | 24, 26 | Find by confirmation request ID; then set `BOOKING_IN_PROGRESS` only for valid pending row. |
| Confirmation/conflict/failure update | `appointments_pilot` | 32, 36 | Update the known row reference only; persist Tehran timestamp and safe category. |
| Manual handoff append | `manual_handoff_queue` | 35 | Lookup `handoff_id`; append only if absent. |

Because Google Sheets has no transactional unique constraint or row lock, implementation must run the specified confirmation critical section serialized per calendar. Before activating the pilot, test that selected n8n Sheets node operations can find and update a precise row without creating accidental duplicate rows.

## 5. Required Credentials

| Credential name (proposed) | Type | Used by | Required scope/control |
| --- | --- | --- | --- |
| `WF001_GOOGLE_CALENDAR_OAUTH` | Google OAuth2 / service identity | Nodes 14, 27, 29, 31, 37 | Read events/free-busy plus create/delete only on allowlisted calendars. |
| `WF001_GOOGLE_SHEETS_OAUTH` | Google OAuth2 / service identity | All Google Sheets nodes | Read/update/append only the dedicated pilot spreadsheet. |
| `WF001_WEBHOOK_AUTH` | Header secret or gateway identity | Node 5 | Verify incoming caller; never logged. |
| `N8N_ENCRYPTION_KEY` | n8n platform secret | n8n runtime | Encrypt stored n8n credentials. |
| `WF001_MATCH_KEY_SECRET` | Runtime secret | Node 4 | Keyed one-way protected matching for National ID/mobile. |

Credential naming is descriptive only; no credential objects are created by this specification.

## 6. Required Environment Variables

Values are environment-specific and must be injected at runtime. Do not place real values in `.env.example`, workflow JSON, Sheets, or logs.

| Variable | Purpose | Example value shape |
| --- | --- | --- |
| `EKMA_ENV` | Environment classification. | `development`, `staging`, `production` |
| `TZ` | n8n process time zone. | `Asia/Tehran` |
| `WF001_TIMEZONE` | Explicit workflow time zone; must equal `Asia/Tehran`. | `Asia/Tehran` |
| `WF001_SPREADSHEET_ID` | Dedicated pilot spreadsheet identifier. | Google Sheet ID |
| `WF001_PATIENTS_SHEET_NAME` | Patient worksheet allowlist name. | `patients_pilot` |
| `WF001_APPOINTMENTS_SHEET_NAME` | Appointment worksheet allowlist name. | `appointments_pilot` |
| `WF001_AUDIT_SHEET_NAME` | Audit worksheet allowlist name. | `audit_log` |
| `WF001_HANDOFF_SHEET_NAME` | Handoff worksheet allowlist name. | `manual_handoff_queue` |
| `WF001_MATCH_KEY_SECRET` | National-ID/mobile protected matching secret. | secret reference/value |
| `WF001_WEBHOOK_AUTH_SECRET` | Incoming webhook verification secret if not handled by gateway. | secret reference/value |
| `WF001_OFFER_TTL_MINUTES` | Offer expiry duration. | bounded positive integer |
| `WF001_DEFAULT_SLOT_MINUTES` | Default appointment duration. | approved integer |
| `WF001_SLOT_BUFFER_MINUTES` | Buffer before/after appointments. | approved integer |
| `WF001_MAX_SLOTS_PER_OFFER` | Response-size and choice bound. | approved integer |
| `WF001_CALENDAR_ALLOWLIST_JSON` | Doctor-key-to-calendar/rules mapping. | protected JSON configuration reference |
| `WF001_WEBHOOK_BASE_URL` | Public HTTPS base URL for callback/response construction where needed. | HTTPS URL |
| `WF001_RETRY_MAX_ATTEMPTS` | Bounded retry count. | `2` |
| `WF001_RETRY_BASE_DELAY_MS` | Initial exponential-backoff delay. | bounded integer |
| `WF001_HANDOFF_RESPONSE_MESSAGE` | Approved generic handoff message. | localized text reference |

Calendar allowlist configuration must contain only approved calendar IDs, display labels, locations, working windows, and allowed slot durations. It must not contain patient data or credentials.

## 7. n8n Expressions and Data Mapping

The following are expression patterns, not workflow JSON. Exact property paths may be adjusted only if the webhook payload contract remains unchanged.

| Use | Expression / mapping pattern |
| --- | --- |
| Action | `{{$json.body.action}}` |
| Request ID | `{{$json.body.request_id}}` |
| Full name | `{{$json.normalized.full_name}}` |
| Protected patient keys | `{{$json.normalized.national_id_match_key}}`, `{{$json.normalized.mobile_match_key}}` |
| Correlation ID | `{{$json.ctx.correlation_id}}` |
| Execution ID | `{{$execution.id}}` |
| Workflow time zone | `{{$env.WF001_TIMEZONE}}` |
| Tehran timestamp | `{{$now.setZone($env.WF001_TIMEZONE).toISO()}}` |
| Spreadsheet ID | `{{$env.WF001_SPREADSHEET_ID}}` |
| Offer expiry | `{{$now.setZone($env.WF001_TIMEZONE).plus({ minutes: Number($env.WF001_OFFER_TTL_MINUTES) }).toISO()}}` |
| Calendar ID from allowlist | `{{$json.calendar_config.calendar_id}}` |
| Selected slot | `{{$json.normalized.selected_slot_start}}` |
| Registration operation key | Derived in node 4 from action, request ID, and safe request fingerprint; thereafter `{{$json.keys.registration_operation_key}}`. |
| Calendar operation key | Derived after valid confirm from booking token reference and confirmation request ID; thereafter `{{$json.keys.calendar_operation_key}}`. |
| Deterministic Calendar event ID | Derived in Code node from calendar operation key according to Google event-ID constraints; thereafter `{{$json.keys.calendar_event_id}}`. |
| Audit event ID | Derived from execution ID plus event phase; thereafter `{{$json.keys.audit_event_id}}`. |

Never use n8n expressions to print raw National ID, raw phone number, OAuth data, match-key secret, full request payload, or booking token into logs, error messages, Calendar content, or audit rows.

## 8. Retry and Reconciliation Behavior

Only transient categories—provider `429`, `5xx`, network timeout, and explicitly documented retryable errors—may retry. The retry sequence is two attempts after the initial action, using exponential backoff based on `WF001_RETRY_BASE_DELAY_MS`, with jitter. Authentication/authorization failures, validation failures, `4xx` policy errors, and data conflicts never retry automatically.

- Read availability: retry transiently, then fail with no stale booking decision.
- Sheets write before Calendar create: retry transiently, then fail closed.
- Calendar event create: on any ambiguous outcome, go to node 29 (retrieve by deterministic ID). Create only if retrieval proves the event does not exist.
- Sheets confirmation/audit after Calendar create: retry transiently, then execute rollback nodes 29/37.
- Rollback delete: retry transiently only after verifying event ownership; unknown result becomes urgent manual handoff.
- Caller retries are absorbed by registration/confirmation operation keys and return the original safe result.

## 9. Rollback Behavior

The only compensating rollback is for a verified Calendar event that cannot be durably represented in `appointments_pilot` and `audit_log`.

1. Retry Sheets confirmation update and terminal audit append within their bounded policies.
2. Retrieve the event by deterministic ID and verify the event marker matches the current calendar operation.
3. Delete only that event, then verify it is absent.
4. If verified deleted, write `ROLLBACK` audit evidence when Sheets becomes available and return a non-confirmation/handoff result.
5. If deletion fails or remains unknown, stop automation, invoke operational alert, create/retry manual handoff, and respond “confirmation pending.”

No rollback deletes an event that does not carry the expected operation marker. No workflow path creates a replacement event automatically.

## 10. Audit Logging

`audit_log` is append-only in the pilot. A start event is written at node 3 before external booking actions; a terminal event is written on every completed, failed, handoff, conflict, and rollback branch. Audit rows use deterministic `audit_event_id` values, making retried append operations safe.

Audit values are limited to execution/correlation IDs, operation references, node/stage, timestamps in `Asia/Tehran`, safe outcome/error codes, retry count, and protected internal row references. They contain no raw identifiers or request content. If start-audit persistence fails, no Calendar/booking action runs.

## 11. Manual Handoff

Node 35 writes one idempotent case to `manual_handoff_queue`, then node 38 records terminal audit evidence and node 40 returns a safe response. Handoff reasons include:

- invalid or unsupported registration/confirmation input requiring staff assistance;
- National-ID/mobile identity conflict or suspected duplicate;
- unallowlisted doctor/calendar or configuration mismatch;
- expired or inconsistent pending booking;
- unknown Calendar-create state;
- failed or unknown Calendar rollback; and
- clinical, urgent, or out-of-scope content.

The handoff case carries only correlation ID, reason code, priority, safe patient/booking reference, and Tehran timestamp. It must never include raw National ID, raw phone number, or full request content.

## 12. Implementation Checklist

- [ ] Project owner approves n8n JSON generation after reviewing this document.
- [ ] A non-production Google account, allowlisted test calendar, and dedicated test spreadsheet are approved.
- [ ] All four worksheets and exact headers are created manually/through approved setup: `patients_pilot`, `appointments_pilot`, `audit_log`, `manual_handoff_queue`.
- [ ] Google OAuth/service identities are configured with verified minimum access.
- [ ] Webhook authentication and public HTTPS routing are configured and tested.
- [ ] n8n `TZ` and `WF001_TIMEZONE` are both `Asia/Tehran`.
- [ ] Environment variables are injected securely; no secret is present in an export or Sheet.
- [ ] Calendar allowlist, work hours, buffers, durations, locale messages, and offer TTL are approved by the clinic.
- [ ] The deployed Calendar integration can use deterministic client event IDs; otherwise the approved OAuth-authenticated HTTP Request design is selected.
- [ ] Confirmation execution serialization/lock is configured and load-tested.
- [ ] Every Sheets operation has lookup/upsert behavior keyed by its deterministic operation ID.
- [ ] Every outcome branch reaches terminal audit and safe response nodes.
- [ ] Calendar create timeout, Sheets-after-Calendar failure, and unknown rollback are tested with synthetic faults.
- [ ] Duplicate identity, duplicate request ID, and same-slot concurrency tests pass.
- [ ] Authorized staff, handoff ownership, rollback runbook, and alert route are named before activation.
- [ ] Project owner reviews generated JSON before import and activation.

## Approval Gate

No n8n workflow JSON, credential, Google Sheet, Calendar configuration, or webhook is created by this document. Explicit project-owner approval is required before JSON generation.
