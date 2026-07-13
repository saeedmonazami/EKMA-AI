# Contributor Instructions

This repository is in its foundation phase. These instructions apply to every human and AI contributor.

## Required approval gate

Do **not** create, modify, import, deploy, or activate:

- application code;
- n8n workflows or workflow JSON exports;
- production infrastructure configuration;
- external integrations, credentials, webhooks, or API calls;
- database migrations or schema implementation;

without explicit approval from the project owner in the current task.

Documentation and project-foundation changes are allowed when requested, provided they contain no secrets and do not bypass this gate.

## Working practices

1. Read `PROJECT_RULES.md` and relevant files in `docs/` before making a change.
2. Keep changes small, reviewable, and scoped to the request.
3. Never commit credentials, patient data, production exports, or generated runtime data.
4. Treat every patient-facing path as safety-critical: provide administrative support only, never diagnosis, triage, treatment, or emergency guidance.
5. Preserve auditability for future workflow, booking, notification, and consent actions.
6. Update documentation alongside any approved architectural decision.

## Decision hierarchy

1. Applicable law, privacy obligations, and patient safety
2. Explicit project-owner instruction
3. `PROJECT_RULES.md`
4. Architecture and operational documentation
5. Local implementation convenience
