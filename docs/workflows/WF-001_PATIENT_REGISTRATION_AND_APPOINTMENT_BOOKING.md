# WF-001 — Patient Registration and Appointment Booking

**Status:** Design approved for planning only; n8n JSON generation and activation require explicit project-owner approval.  
**Scope:** Google Calendar, Google Sheets, and n8n only.  
**Explicitly excluded:** PostgreSQL, WhatsApp, voice, SMS, email, payments, RAG, and all other workflow catalog entries.

## 1. Purpose and Boundary

WF-001 receives minimum patient registration data, detects an existing patient using National ID and mobile-number match keys, validates the request, checks availability in a selected doctor’s Google Calendar, offers eligible slots, receives an explicit selection/confirmation, creates a Google Calendar event, saves the patient and booking records to Google Sheets, and returns a confirmation message through the initiating HTTP/API channel.

This is an administrative booking workflow. It must not collect clinical history, provide diagnosis, triage symptoms, give medical advice, or infer that a patient’s request is urgent. Such content must be rejected safely and directed to human staff through the configured operational process.

### Implementation shape

This is **one n8n workflow** with one webhook entry point and two request actions:

1. `register` — validate the registration request, calculate available slots, save a short-lived pending booking record in Google Sheets, and return the offer.
2. `confirm` — validate the selected offered slot, re-check Google Calendar immediately before booking, create the calendar event, update the Sheets row, and return the booking confirmation.

The two-step design is mandatory because an available slot can become unavailable between the offer and the patient’s confirmation. A slot is never considered booked until Google Calendar has confirmed event creation.

## 2. Workflow Contract

### Trigger

An authenticated HTTPS POST request to the approved n8n webhook endpoint. The calling interface is out of scope; it may be a controlled internal form or later an approved channel adapter. The webhook must not be publicly usable without a suitable authentication/control mechanism approved for the deployment.

### Time-zone rule

`Asia/Tehran` is mandatory for every workflow timestamp, scheduling calculation, offer expiry, Google Calendar event start/end value, Google Sheets value, audit event, and patient-facing response. Timestamps must include their UTC offset; UTC may be retained only as an additional technical reference, never as the sole displayed or operational value. The calendar configuration must be explicitly set to the same time zone rather than inheriting an n8n server default.

### Request: `register`

| Field | Required | Rules |
| --- | --- | --- |
| `action` | Yes | Literal `register`. |
| `request_id` | Yes | Caller-generated idempotency key; unique for the operation. |
| `full_name` | Yes | Trimmed, non-empty, within approved length limit. |
| `national_id` | Yes | Iranian National ID: normalize to 10 ASCII digits and validate its approved format/checksum. Used in memory only to derive a protected deterministic match key; the raw value is never written to Sheets, logs, Calendar, or responses. |
| `phone` | Yes | Normalize to the approved Iranian/international format. It is used in memory to derive a protected deterministic match key; the protected Sheet may retain the normalized value only if the approved privacy policy requires it. |
| `doctor_calendar_id` | Yes | Must be in the approved calendar allowlist; never accept arbitrary Calendar IDs. |
| `appointment_date` | Yes | Date interpreted and stored in the mandatory operational time zone, `Asia/Tehran`. |
| `slot_duration_minutes` | No | If supplied, must be an approved duration; otherwise use configured default. |
| `locale` | No | Supported locale; default to organization setting. |

Optional fields must be minimized. No clinical details, free-text symptoms, payment information, or attachments are accepted by this workflow. National ID is accepted solely for duplicate detection and is immediately transformed into a protected match key.

### Response: `register`

Success returns a short-lived `booking_token`, `offer_expires_at`, and an ordered list of available slot start/end times in the configured time zone. The response must clearly state that the slot is **not reserved** until confirmed.

### Request: `confirm`

| Field | Required | Rules |
| --- | --- | --- |
| `action` | Yes | Literal `confirm`. |
| `request_id` | Yes | New caller-generated idempotency key for the confirmation operation. |
| `booking_token` | Yes | Must identify one active, unexpired pending Sheets row. |
| `selected_slot_start` | Yes | Must exactly match one offered slot for the pending request. |
| `confirmation` | Yes | Explicit boolean/affirmative value. |

### Response: `confirm`

On success, return appointment status, date/time/time zone, doctor display label, location/contact instructions if configured, and a safe appointment reference. It must not expose Google Calendar IDs or unnecessary patient data. On failure/conflict, return a safe status and, where appropriate, new availability options or a staff-contact message.

## 3. Data Stores and Google Sheets Model

Google Sheets is a temporary operational register, not a long-term patient database. Access is restricted to authorized operational staff. The sheet must be in a dedicated approved Google account/drive and must not be publicly shared.

### Protected matching design

National ID and mobile number are personal/sensitive identifiers. The workflow derives `national_id_match_key` and `mobile_match_key` in n8n using an approved, environment-held keyed one-way transformation. The same normalized value always yields the same key in the same environment, enabling exact matching without storing a raw National ID. The key material is an n8n secret and is never placed in the sheet, workflow export, logs, or calendar event.

Existing-patient decision rules:

1. Both match keys resolve to the same active `patient_pilot_id`: reuse that patient record; do not create a duplicate registration.
2. Neither key resolves: create exactly one new patient record.
3. Only one key resolves, or the two keys resolve to different records: create an `IDENTITY_CONFLICT` handoff; do not modify, merge, or expose either patient record.
4. A duplicate registration request ID returns the original result. A same-identity request with a different registration request ID reuses the existing patient record and may create a separate appointment attempt only after all booking checks pass.

### Required worksheet: `patients_pilot`

Each row is the one reusable pilot patient record. Required columns:

| Column | Purpose |
| --- | --- |
| `patient_pilot_id` | Opaque internal patient reference; unique. |
| `national_id_match_key` | Protected deterministic National-ID lookup key; unique among active records. |
| `mobile_match_key` | Protected deterministic mobile-number lookup key; unique among active records under the approved duplicate policy. |
| `full_name` | Minimum patient display name. |
| `phone` | Normalized phone only if approved by the privacy policy; never used as the primary key. |
| `status` | `ACTIVE`, `PENDING_REVIEW`, or `INACTIVE`. |
| `created_at` / `updated_at` | Asia/Tehran local timestamps with offset. |
| `correlation_id` | Safe trace identifier for the latest material change. |

### Required worksheet: `appointments_pilot`

Each row represents one registration/booking attempt. Required columns:

| Column | Purpose |
| --- | --- |
| `booking_token` | Random opaque token for the pending interaction; unique. |
| `registration_request_id` | Idempotency key for `register`. |
| `confirmation_request_id` | Idempotency key for `confirm`, when received. |
| `status` | `PENDING_CONFIRMATION`, `CONFIRMED`, `EXPIRED`, `CONFLICT`, `FAILED`, or `CANCELLED`. |
| `full_name` | Minimum patient display name. |
| `patient_pilot_id` | Reference to the matching row in `patients_pilot`. |
| `national_id_match_key` | Protected key copied only where needed for reconciliation; never raw National ID. |
| `mobile_match_key` | Protected mobile lookup key. |
| `phone` | Normalized phone only if approved by the privacy policy. |
| `doctor_calendar_key` | Approved internal key/label, not necessarily raw external Calendar ID. |
| `requested_date` | Requested local date. |
| `offered_slots` | Protected serialized list or safe reference to offered slots. |
| `selected_slot_start` | Selected local/UTC instant after confirmation. |
| `selected_slot_end` | Calculated end instant. |
| `time_zone` | Decision time zone. |
| `offer_expires_at` | Time after which confirmation is rejected. |
| `calendar_event_reference` | Google event reference after verified creation. |
| `created_at` | Registration timestamp. |
| `confirmed_at` | Confirmation timestamp where applicable. |
| `correlation_id` | Safe trace identifier. |
| `last_error_category` | Redacted operational error category only. |

### Required worksheet: `audit_log`

Append-only workflow audit evidence. Required columns: `audit_event_id`, `workflow_id`, `execution_id`, `correlation_id`, `request_action`, `event_phase` (`STARTED`, `COMPLETED`, `FAILED`, `HANDOFF`, `ROLLBACK`), `outcome`, `safe_error_category`, `patient_pilot_id` where available, `booking_token_reference` where justified, `calendar_operation_key`, `timestamp_tehran`, and `retry_count`. It contains no raw National ID, phone number, booking token, request payload, OAuth data, or free-form sensitive content.

### Required worksheet: `manual_handoff_queue`

Records a validation/identity/recovery case for human staff. Required columns: `handoff_id`, `correlation_id`, `reason_code`, `priority`, `status`, `safe_patient_reference`, `created_at_tehran`, `owner`, and `resolution_reference`. It stores only the minimum safe context required to resolve the issue.

No credentials, full webhook payloads, raw National IDs, raw clinical text, access tokens, or free-form sensitive information may be written to any worksheet.

## 4. Node-by-Node Design

The names below are logical n8n node names. Exact node type/options are finalized only when JSON generation is approved.

### Shared entry and safety nodes

| # | Node | Responsibility | Success path | Failure path |
| --- | --- | --- | --- | --- |
| 1 | `Webhook — Receive Booking Request` | Accept HTTPS POST; capture request and minimal request metadata. | Pass normalized request payload onward. | Return generic `400`/`401`; log safe category. |
| 2 | `Set — Create Correlation Context` | Create correlation ID, execution ID, and timestamps rendered in `Asia/Tehran` with offset; set the workflow time zone explicitly to `Asia/Tehran`. | Pass context. | Stop safely if context cannot be created. |
| 3 | `Google Sheets — Audit Log: Execution Started` | Append an idempotent `STARTED` audit event for every workflow execution before any external booking action. | Pass context. | Fail closed and return safe temporary error; do not call Calendar. |
| 4 | `Code/Set — Normalize and Protect Input` | Trim text; normalize phone and National ID; validate National-ID checksum; derive protected deterministic match keys; normalize all date/times to `Asia/Tehran`; reject unsupported keys/content. | Produce normalized request with match keys, never raw National ID downstream. | Route to validation handoff without persisting raw sensitive input. |
| 5 | `IF — Validate Webhook Authorization` | Verify the approved authentication/control mechanism. | Continue. | Audit terminal failure and return `401`/`403`. |
| 6 | `Switch — Route Action` | Route only `register` or `confirm`. | Enter the relevant branch. | Route to validation handoff and return safe `400`. |

### `register` branch: registration and slot offer

| # | Node | Responsibility | Success path | Failure path |
| --- | --- | --- | --- | --- |
| 7 | `IF — Validate Registration Fields` | Require and validate name, National ID, mobile, allowlisted doctor, date, and request ID; reject unsafe free text. | Continue. | Route to manual validation handoff. |
| 8 | `Google Sheets — Lookup Registration Request` | Find an existing row by `registration_request_id` before any new external action. | Return prior idempotent result or continue. | Treat sheet outage as temporary failure; do not continue. |
| 9 | `Google Sheets — Lookup Existing Patient` | Look up active patient records by both protected National-ID and mobile match keys. | Continue with duplicate decision. | Treat sheet outage as temporary failure; do not continue. |
| 10 | `IF — Resolve Patient Identity` | Reuse same matched patient; create new patient if no match; route one-key/different-record matches to `IDENTITY_CONFLICT` handoff. | Continue with one `patient_pilot_id`. | Manual human handoff; never auto-merge. |
| 11 | `Google Sheets — Upsert Patient Record` | Idempotently create a `patients_pilot` row only for a new identity, using unique match keys and patient operation key. | Continue. | Fail closed; no Calendar action. |
| 12 | `Set — Resolve Approved Calendar Configuration` | Map approved doctor key to calendar ID, display label, working hours, slot duration, and location. | Continue with resolved configuration. | Route to manual validation handoff; do not query arbitrary calendar. |
| 13 | `Google Calendar — Query Busy Events` | Idempotently query the approved calendar for the requested local-day availability window, using Tehran-local boundaries. | Return busy periods. | Return temporary calendar-unavailable response. |
| 14 | `Code — Calculate Eligible Slots` | Subtract busy periods from configured Tehran-local working hours; enforce duration, buffers, lead time, and max slots. | Produce ordered offered slots. | Return no-availability or calculation failure safely. |
| 15 | `IF — Slots Available` | Ensure at least one slot remains. | Continue. | Return `NO_AVAILABLE_SLOTS` with permitted next-step guidance. |
| 16 | `Set — Create Idempotent Pending Booking Record` | Create opaque booking token, offer expiry, `PENDING_CONFIRMATION`, and unique pending-record operation key. | Continue. | Stop if token/context missing. |
| 17 | `Google Sheets — Upsert Pending Record` | Persist registration and offer idempotently before returning it. | Continue. | Return temporary storage failure; no booking is made. |
| 18 | `Audit Log — Execution Completed` | Append an idempotent terminal audit event with outcome `SLOTS_OFFERED`. | Continue. | Return safe failure; no Calendar event exists. |
| 19 | `Respond to Webhook — Offer Slots` | Return token, expiry, available slots, and unreserved-slot notice. | End. | N/A. |

### `confirm` branch: confirmation and booking

| # | Node | Responsibility | Success path | Failure path |
| --- | --- | --- | --- | --- |
| 20 | `IF — Validate Confirmation Fields` | Validate token, selection, explicit confirmation, and confirmation request ID. | Continue. | Route to manual validation handoff. |
| 21 | `Google Sheets — Lookup Pending Booking` | Retrieve row by booking-token lookup key. | Continue with exactly one record. | Route invalid/unknown token to manual handoff without leaking details. |
| 22 | `IF — Validate Pending State` | Require `PENDING_CONFIRMATION`, unexpired offer, matching offered slot, and allowed calendar configuration. | Continue. | Route expired/invalid selection to manual handoff or fresh-registration response. |
| 23 | `Google Sheets — Lookup Confirmation Request` | Detect duplicate confirm call by confirmation operation key before Calendar action. | Return prior confirmed result if completed; otherwise continue. | Treat sheet outage as temporary failure. |
| 24 | `Set — Reserve Booking Operation` | Persist an idempotent `BOOKING_IN_PROGRESS` state with deterministic calendar operation key before Calendar create. | Continue. | Stop; no Calendar action. |
| 25 | `Google Calendar — Re-check Busy Events` | Re-query the exact selected interval immediately before creation. | Continue only if free. | Update row to `CONFLICT`; return safe availability response. |
| 26 | `IF — Selected Slot Still Free` | Guard the create operation against calendar conflict. | Continue. | Update Sheets conflict state; return alternatives only if safely recomputed. |
| 27 | `Google Calendar — Create or Retrieve Appointment Event` | Create using a deterministic client event ID/operation key; on retry or ambiguous result, retrieve the same event before any create attempt. Use a native Calendar node only if it supports this; otherwise use an approved authenticated Calendar API request in n8n. | Receive verified created event reference. | Enter unknown-outcome reconciliation; never create a second event blindly. |
| 28 | `Google Sheets — Update Confirmed Booking` | Idempotently set `CONFIRMED`, selection, event reference, confirmation key, and Tehran timestamps. | Continue. | Enter rollback sequence. |
| 29 | `Audit Log — Execution Completed` | Append idempotent terminal audit result `CONFIRMED`, including only safe references. | Continue. | Enter rollback sequence because durable audit evidence is missing. |
| 30 | `Respond to Webhook — Confirmation Message` | Return confirmed Tehran-local date/time, status, and safe reference. | End. | N/A. |

### Error and recovery nodes

| # | Node | Responsibility |
| --- | --- |
| 31 | `Google Sheets — Create Manual Handoff` | Idempotently create a validation, identity-conflict, or recovery handoff record with a safe reason code. |
| 32 | `Google Sheets — Update Failure/Conflict State` | Idempotently record only safe status/error category, correlation ID, Tehran timestamp, and retry/review state. |
| 33 | `Google Calendar — Roll Back Created Event` | Delete only the deterministic event created by this workflow after post-create Sheets/audit persistence failure, following the rollback rules below. |
| 34 | `Audit Log — Execution Terminal Event` | Append an idempotent `FAILED`, `HANDOFF`, or `ROLLBACK` event for every terminal path. |
| 35 | `Error Trigger / Error Workflow — Operational Alert` | Receive workflow failures, redact data, notify the approved operator channel, and preserve correlation ID. This is a design dependency only; it is not created by WF-001. |
| 36 | `Respond to Webhook — Safe Error or Handoff Response` | Provide a non-sensitive response with a correlation/reference ID and staff-contact route. |

## 5. Required Credentials and Access

Credentials are created and stored in n8n’s encrypted credential store or another approved secret mechanism only. They are never placed in workflow JSON, Google Sheets, source control, logs, or documentation.

| Credential | Used by | Minimum scope / controls |
| --- | --- | --- |
| Google Calendar OAuth2 or approved service identity | Nodes 13, 25, 27, 33 | Read free/busy/events and create/delete only workflow-owned events for allowlisted doctor calendars; no broad calendar access. |
| Google Sheets OAuth2 or approved service identity | All Sheets lookup/upsert/audit/handoff nodes | Read/append/update only the dedicated pilot spreadsheet and its approved worksheets; no Drive-wide access beyond what is unavoidable for the chosen auth method. |
| Webhook authentication secret or gateway identity | Node 5 | Stored outside payload/logs; rotate under the project’s secrets policy. |
| n8n encryption key | n8n platform | Required to encrypt stored credentials; managed outside the workflow. |
| Protected match-key secret | Input protection node | Keyed one-way National-ID/mobile matching; stored only as a runtime secret and rotated through an approved migration plan. |

The calendar allowlist, working hours, buffers, appointment durations, locations, and doctor labels are approved configuration—not patient-provided data and not secrets.

## 6. Error Handling

| Condition | Workflow behavior | Patient-facing outcome | Operator action |
| --- | --- | --- | --- |
| Invalid/missing fields | Reject before Calendar/Sheets access. | Explain required field correction without echoing sensitive data. | None unless recurring. |
| Validation failure requiring assistance | Create an idempotent `manual_handoff_queue` case and terminal audit event. | State that staff will assist; expose only correlation/reference ID. | Resolve the validation case manually. |
| National ID/mobile duplicate conflict | Create `IDENTITY_CONFLICT` handoff; do not register, merge, or disclose matching records. | State that staff must verify registration details. | Verify identity and resolve under privacy policy. |
| Unauthorized webhook | Reject and audit safe security metadata. | Generic unauthorized response. | Investigate repeated attempts. |
| Unknown doctor/calendar | Reject against allowlist. | Ask for an approved doctor/service selection. | Review configuration if expected. |
| No available slots | Do not create a pending record. | State that no slots are available for the request. | Staff option per clinic policy. |
| Offer expired/slot mismatch | Do not create event. Mark pending row expired where appropriate. | Ask the patient to request new availability. | None unless recurring. |
| Selected slot became busy | Update row to `CONFLICT`; never create event. | Say the slot is no longer available and offer a new search path. | Review if frequent. |
| Calendar create timeout/unknown result | Retrieve by deterministic Calendar event ID/operation key before any create retry; mark row review-needed if state remains unknown. | Say confirmation is pending; provide safe reference. | Resolve reconciliation case. |
| Sheets write fails before Calendar create | Stop; no event is created. | Temporary service issue. | Investigate Sheets availability. |
| Sheets/audit persistence fails after Calendar create | Run the bounded rollback sequence; do not claim confirmation. If rollback cannot be verified, create a recovery handoff/alert using the error path. | State that booking confirmation is pending and staff will assist. | Resolve rollback/reconciliation case immediately. |
| Duplicate register request | Return existing valid offer; do not append a second row. | Same offer/status. | Investigate only if abuse pattern. |
| Duplicate confirm request | Return existing confirmed result; do not create a second event. | Same confirmation/status. | Investigate if mismatched payload. |
| Clinical/urgent content | Do not process booking details beyond safe minimum. | Administrative limitation and staff/emergency guidance approved by clinic policy. | Priority staff handling. |

## 7. Retry Strategy and Idempotency

### Idempotency

- `registration_request_id` makes `register` repeat-safe. The same valid request returns the existing unexpired offer rather than appending another row.
- `confirmation_request_id` makes `confirm` repeat-safe. The completed result is returned rather than creating another event.
- `patient_operation_key`, `pending_booking_operation_key`, `calendar_operation_key`, `handoff_id`, and `audit_event_id` are deterministic per logical external action. Each Sheets append/upsert first checks its operation key and returns the pre-existing result if found.
- `booking_token` is opaque, single-use after confirmation, expires after the configured offer TTL, and is not derived from patient data.
- Google Calendar event creation uses a deterministic client event ID derived from `calendar_operation_key`, subject to Google Calendar ID requirements. The workflow retrieves that event before retrying an ambiguous create. The event contains only a safe correlation/reference marker, never unnecessary PHI.
- The pilot deployment must serialize competing confirmation executions for the same doctor/calendar (or use an approved equivalent lock) from final free/busy check through Calendar create. This is required because Google Sheets does not provide transactional row locking.
- Every external action includes an explicit operation key, a pre-action lookup where supported, a bounded retry policy, and a terminal audit event. An action whose outcome is unknown is reconciled before retry, never repeated blindly.

### Retries

| Operation | Retry policy | Guardrail |
| --- | --- | --- |
| Google Calendar availability reads | Up to 2 bounded retries with exponential backoff for transient provider/network errors. | Never use stale availability to book. |
| Google Sheets read/write before event create | Up to 2 bounded retries for transient errors. | Stop if persistence remains unavailable. |
| Google Calendar event creation | No blind automatic retry after timeout or ambiguous response. | Reconcile first using correlation/provider metadata. |
| Google Sheets update after event creation | Up to 2 bounded retries; then create a reconciliation alert/task. | Never create another event to compensate. |
| Google Calendar delete during rollback | Up to 2 bounded retries only after verifying event ownership and deterministic ID. | If delete outcome is unknown, stop and require urgent manual reconciliation. |
| Google Sheets audit/handoff append | Up to 2 bounded retries using the deterministic event/case ID. | If terminal audit cannot be persisted after Calendar create, start rollback; if before create, fail closed. |
| Webhook/caller retry | Supported through request IDs and idempotent responses. | Mismatched payload for same ID is rejected and logged as a safe integrity event. |

Retry delays, limits, and provider-specific error classification must be configurable and tested with synthetic failures.

### Rollback: Calendar succeeded, Sheets failed

1. After a verified Calendar event creation, retry the required Sheets `CONFIRMED` update up to two times using the same operation key.
2. If the booking update still fails, retry the terminal audit append up to two times. If durable Sheets state remains unavailable, do **not** return a confirmation or retry Calendar creation.
3. Retrieve the Calendar event by its deterministic event ID and verify it belongs to the current `calendar_operation_key`.
4. Delete that specific event, then verify deletion. This compensating delete is the rollback action.
5. If deletion is verified, return a safe failure/handoff response and write `ROLLBACK` audit evidence when Sheets becomes available. The patient is not told that an appointment was confirmed.
6. If deletion fails or its outcome is unknown, stop automated changes, issue an urgent operator alert with correlation ID and safe Calendar operation reference, and create/retry a manual handoff record. The patient receives only “confirmation pending; staff will contact you.”
7. Under no condition may the workflow create a second Calendar event as a repair action.

## 8. Logging and Audit Events

Log structured, redacted events only. Required fields: `timestamp_tehran` (with `Asia/Tehran` offset), workflow ID (`WF-001`), execution ID, correlation ID, action, stage/node, organization/doctor configuration key, outcome, safe error category, duration, and retry count.

Do not log full name, National ID, National-ID/mobile match keys, phone number, raw request body, booking token, OAuth data, Calendar event body, Google Sheet row content, or complete message text. Where a patient reference is necessary for investigation, use a protected internal/sheet row reference accessible only to authorized staff.

Minimum events:

- request accepted/rejected and validation outcome;
- execution `STARTED` and terminal `COMPLETED`, `FAILED`, `HANDOFF`, or `ROLLBACK` audit events for every execution;
- existing-patient lookup outcome, without identifiers;
- availability lookup start/result/error;
- pending record created or reused;
- confirmation validated/expired/conflicted;
- calendar event create attempted/confirmed/unknown;
- sheet state update outcome;
- duplicate request detected;
- final response status; and
- error alert/reconciliation required.

Google Sheets `audit_log` serves as the pilot audit record. n8n execution logs are not a substitute for it and must follow the configured retention/redaction policy. If the audit log is unavailable before a Calendar action, the workflow fails closed.

## 9. Testing Checklist

All tests use synthetic data and a non-production Google Calendar and Google Sheet.

### Functional tests

- [ ] Valid `register` returns correctly calculated slots in `Asia/Tehran`.
- [ ] Every persisted timestamp, Calendar event time, offer expiry, and response time is rendered/recorded using `Asia/Tehran` with the correct offset, including daylight-saving policy validation.
- [ ] A National ID and mobile that match one existing patient reuse that `patient_pilot_id` without creating a duplicate patient row.
- [ ] No matching National ID/mobile creates exactly one patient row, even when the request is retried.
- [ ] One-key-only and different-record National-ID/mobile matches create an `IDENTITY_CONFLICT` handoff and do not merge records.
- [ ] Working hours, buffer, duration, lead time, and day-boundary rules are applied.
- [ ] No available slots returns the correct safe result and creates no booking row/event.
- [ ] Valid `confirm` creates exactly one Google Calendar event and updates exactly one sheet row.
- [ ] Confirmation response contains the correct time, timezone, and safe reference.
- [ ] Duplicate `register` with the same request ID returns the original offer.
- [ ] Duplicate `confirm` with the same request ID returns the original confirmed result without another event.
- [ ] Every external Sheets/Calendar/audit/handoff operation returns the same result when repeated with its operation key.
- [ ] Expired token and mismatched selected slot are rejected.
- [ ] A second simulated booking of the offered slot produces a conflict rather than duplicate booking.

### Validation and security tests

- [ ] Missing, malformed, oversized, and unsupported fields are rejected.
- [ ] Invalid National IDs, invalid National-ID checksum, and invalid mobile values route to manual handoff when configured for staff review.
- [ ] Every validation-failure path creates exactly one safe manual-handoff record and exactly one terminal audit event.
- [ ] Arbitrary/unallowlisted calendar IDs cannot be queried.
- [ ] Unauthorized and malformed webhook calls are rejected without information leakage.
- [ ] Free-text clinical/urgent content routes to the safe escalation response.
- [ ] Secrets, raw phone numbers, tokens, and full payloads are absent from logs and error output.
- [ ] Raw National ID and all match keys are absent from Sheets columns other than approved protected match-key fields, logs, Calendar event content, and responses.
- [ ] Google credentials cannot access unrelated calendars or spreadsheets beyond approved scope.

### Resilience and recovery tests

- [ ] Calendar availability read transient failure observes the bounded retry policy.
- [ ] Sheets failure before event creation leaves no event.
- [ ] Calendar create timeout enters reconciliation state and does not create a duplicate on retry.
- [ ] Sheets failure after event creation triggers reconciliation and does not create a duplicate event.
- [ ] Sheets `CONFIRMED` update failure after verified Calendar creation retries, deletes only the owned deterministic Calendar event, verifies rollback, and returns no false confirmation.
- [ ] A failed/unknown Calendar rollback triggers an urgent manual handoff and does not issue a second Calendar create/delete attempt beyond the bounded policy.
- [ ] Every execution, including unauthorized, validation failure, conflict, and rollback paths, produces a safe `audit_log` event or fails closed before an external booking action.
- [ ] Error alert includes correlation ID and safe category only.
- [ ] Manual reconciliation can match a calendar event to its pending/failed sheet row using safe metadata.

### Acceptance conditions before activation

- [ ] Project owner approves generated n8n JSON after review.
- [ ] Calendar allowlist, scheduling rules, offer TTL, and location/confirmation wording are approved by the clinic.
- [ ] Google credential scopes and sharing permissions are verified.
- [ ] Webhook authentication and HTTPS deployment are verified.
- [ ] Named operational owner and manual reconciliation process are documented.
- [ ] Synthetic end-to-end test evidence is recorded and rollback is confirmed.

## 10. Approval Required for Next Step

This document is the design for WF-001 only. It does not create an n8n workflow or JSON export. Explicit approval is required before generating the n8n workflow JSON, configuring Google credentials, creating the Google Sheet, or activating the webhook.
