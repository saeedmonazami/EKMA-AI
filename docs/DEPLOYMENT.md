# Deployment Strategy

## Status

Deployment assets have not been created. Docker Compose and production operations require explicit approval.

## Initial deployment direction

- Run n8n in a self-hosted Docker environment.
- Keep persistent n8n runtime data on managed, backed-up storage.
- Inject secrets at runtime; do not bake them into images or commit them to the repository.
- Start with a dedicated development environment before any staging or production deployment.
- Restrict administrative access and expose only approved inbound endpoints.

## Future Docker Compose services

The planned Compose topology may include n8n, PostgreSQL, a reverse proxy, monitoring, and backup jobs. The exact services, ports, volumes, networks, image versions, and hardening settings remain unimplemented pending approval.

## Environment expectations

| Environment | Purpose | Data rule |
| --- | --- | --- |
| Development | Local experimentation and documentation validation | Synthetic data only |
| Staging | Pre-release integration and operational testing | Synthetic or explicitly approved masked data |
| Production | Live approved services | Real data under approved privacy and security controls |

## Release checklist for future use

- Approved change scope and rollback plan
- Reviewed environment variables and secret injection
- Backup and restore verification
- Health checks, logging, monitoring, and alerting
- Access-control and webhook-security verification
- Synthetic end-to-end test evidence
- Named on-call or operational owner
