# EKMA AI

EKMA AI is a production-oriented AI Medical Receptionist platform for clinics, hospitals, laboratories, and medical centers. It is being founded as an integration-first system: n8n initially coordinates approved automations around Google Calendar and Google Sheets, while the architecture remains ready for PostgreSQL, messaging channels, voice, payments, RAG, monitoring, and backup.

> **Current phase:** project foundation and architecture. No application code or operational workflows have been created.

## Purpose

The platform will help medical organizations receive patient requests, manage appointments, send notifications, and route operational tasks safely. It must preserve patient trust, maintain auditable operations, and avoid making clinical decisions.

## Current stack

- Self-hosted n8n in Docker
- Google Calendar
- Google Sheets
- VS Code
- Codex CLI

## Repository layout

```text
EKMA-AI/
├── docs/                    # Architecture, operational, security, and product documentation
├── infrastructure/          # Reserved for deployment and platform configuration
│   └── docker/              # Reserved for future Docker Compose and container assets
├── n8n/                     # Reserved for approved n8n workflow exports and related documentation
├── .env.example             # Safe environment-variable template
├── .gitignore               # Repository hygiene and secret protection
├── AGENTS.md                # Instructions for human and AI contributors
├── PROJECT_RULES.md         # Engineering and product guardrails
└── README.md                # Project entry point
```

## Documentation

- [Architecture](docs/ARCHITECTURE.md)
- [Roadmap](docs/ROADMAP.md)
- [Data model](docs/DATABASE.md)
- [Workflow principles](docs/WORKFLOWS.md)
- [Security](docs/SECURITY.md)
- [Deployment](docs/DEPLOYMENT.md)

## Contribution boundary

Before adding n8n workflows, integration configurations, or application code, obtain explicit approval from the project owner. See [AGENTS.md](AGENTS.md) and [PROJECT_RULES.md](PROJECT_RULES.md).
