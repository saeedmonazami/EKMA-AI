# Security and Privacy Baseline

## Security objectives

Protect patient and organizational data, prevent unauthorized actions, preserve service availability, and maintain trustworthy audit trails.

## Baseline controls

- Use unique credentials per environment and least-privilege access for each integration.
- Keep secrets out of Git; use environment injection or an approved secrets manager.
- Enforce TLS for all production traffic and secure webhook endpoints with signature verification where supported.
- Protect n8n with strong authentication, restricted administration, encryption keys, and timely updates.
- Apply role-based access control for staff, with access reviews and revocation procedures.
- Log security-relevant events with minimal sensitive content and protect log access.
- Maintain encrypted backups and test restoration on a defined schedule.
- Keep dependencies and base images patched under a documented maintenance process.

## Privacy requirements before production data

Before processing real patient data, document applicable legal and contractual obligations, data-controller/processor responsibilities, lawful basis, consent notices, retention schedule, data-subject request process, incident response, and vendor data-processing terms. Obtain suitable legal and security review.

## Incident response outline

1. Contain the incident and preserve relevant evidence.
2. Assess affected systems, data categories, and scope.
3. Revoke or rotate compromised credentials.
4. Notify the designated project and compliance owners.
5. Meet any applicable notification obligations.
6. Document remediation and validate recovery.

This document is a technical baseline, not legal advice.
