# EKMA AI Project Rules

## Product boundary

EKMA AI is an administrative medical receptionist, not a medical professional. It may assist with scheduling, clinic information, reminders, and approved administrative intake. It must not diagnose, prescribe, assess urgency, interpret test results, or replace emergency services or clinical staff.

## Privacy and data handling

- Apply data minimization: collect only what an approved use case needs.
- Do not place protected health information (PHI), personal data, credentials, or live appointment data in source control, examples, logs, tickets, or prompts.
- Use synthetic, clearly labelled sample data for testing and documentation.
- Define retention, deletion, consent, and access rules before handling production patient data.
- Keep patient identity and appointment data separate where practical; use stable internal identifiers rather than exposing external identifiers.

## Security baseline

- Store secrets only in approved secret-management or environment-injection mechanisms.
- Use least-privilege service accounts and separate development, staging, and production credentials.
- Require authentication, authorization, encryption in transit, and audit logs for production services.
- Verify webhook signatures and validate all inbound data.
- Never log access tokens, credentials, raw PHI, or full message contents unless an approved, protected audit policy requires it.

## Engineering baseline

- Design integrations to be idempotent, retry-safe, observable, and reversible where possible.
- Version control approved workflow exports and infrastructure definitions.
- Review failure paths, duplicate delivery, time zones, localization, and human handoff before release.
- Make no production change without a rollback plan.
- Record material architecture decisions in the relevant documentation before implementation.

## Approval rule

No workflows or application code may be written until the project owner explicitly authorizes that work. Approval for one workflow or component does not automatically approve others.
