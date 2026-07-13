# Roadmap

This roadmap states intended delivery stages; it does not authorize implementation.

## Phase 0 — Foundation (current)

- Repository structure, contribution guardrails, and architecture documentation
- Security and deployment baseline
- Approval gate for code and workflows

## Phase 1 — Validated scheduling pilot

- Confirm clinic operating rules, appointment types, staff handoff, and consent language
- Approve and create a minimal n8n scheduling workflow
- Integrate a single Google Calendar and a controlled Google Sheet register
- Test duplicate handling, errors, time zones, and manual recovery with synthetic data

## Phase 2 — Production data foundation

- Approve PostgreSQL data model, migrations, access controls, backups, and retention policy
- Establish audit events and operational dashboards
- Move operational records from the temporary spreadsheet model to PostgreSQL

## Phase 3 — Multi-channel reception

- Add approved channel adapters: Telegram, Gmail, WhatsApp, Bale Messenger, SMS, voice, and website chat
- Normalize inbound requests and enforce consent and identity rules per channel
- Provide consistent staff handoff and conversation history policies

## Phase 4 — Intelligence and payments

- Add governed RAG over approved non-clinical organizational knowledge
- Add a payment gateway with reconciliation, failure handling, and staff escalation
- Introduce quality evaluation and guarded automation improvements

## Phase 5 — Operations at scale

- Docker Compose deployment, monitoring, alerting, backup verification, and disaster-recovery exercises
- Multi-tenant or multi-organization controls, if approved
- Security review, load testing, and release management

## Exit criteria for each phase

Every phase requires documented acceptance criteria, security review proportional to risk, rollback procedures, synthetic-data testing, and explicit owner approval before production activation.
