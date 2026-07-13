# Architecture

## Architectural intent

EKMA AI will be an integration-first, modular medical reception platform. The initial deployment uses n8n as the orchestration layer with Google Calendar and Google Sheets. Future components must be introduced behind clear boundaries so that data storage, channels, and automation logic can evolve without a rewrite.

## Initial logical components

```text
Patient or staff request
        │
        ▼
Approved communication channel
        │
        ▼
n8n orchestration ─────► Google Calendar (appointment availability/events)
        │
        └──────────────► Google Sheets (temporary operational register)
```

No workflow is implemented yet. The diagram describes the target relationship only.

## Target architecture

```text
Channels (Telegram, WhatsApp, Bale, SMS, voice, web chat, email)
        │
        ▼
Channel adapters / webhook boundary
        │
        ▼
Orchestration and policy layer ───► Human staff handoff
        │
        ├──► Scheduling service / Google Calendar
        ├──► Operational data / PostgreSQL
        ├──► Knowledge retrieval / RAG
        ├──► Notifications / email and messaging providers
        └──► Payments / approved gateway
        │
        ▼
Observability, audit, backups, and monitoring
```

## Core principles

- **Safety by design:** administrative assistance only; uncertain or sensitive requests route to staff.
- **Integration boundaries:** providers are replaceable and isolated from business policy.
- **System of record:** Google Sheets is transitional; PostgreSQL becomes the operational source of truth when approved.
- **Human-in-the-loop:** appointment exceptions, identity uncertainty, complaints, clinical questions, and high-risk actions require staff escalation.
- **Auditability:** significant actions must later have correlation IDs, actor/channel information, timestamps, and outcomes.
- **Resilience:** retries, idempotency keys, dead-letter handling, and duplicate-event protection are required before production use.

## Environments

Development, staging, and production must be separated by credentials, data, and deployment configuration. Production data must never be used in development without a documented, approved exception.
