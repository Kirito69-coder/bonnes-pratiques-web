# Deployment checklist

This reference lists deployment and release items to verify before going live or after a deployment.

## Pre-deployment

- Changes and migrations reviewed.
- Secrets confirmed to come from the target environment.
- Tests, linters and dependency checks run.
- Backups and restore procedure confirmed.
- Rollback plan documented.
- DNS, certificates and configuration changes confirmed.

## After deployment

- Minimal functional checks on critical paths.
- Logs, 4xx/5xx errors, latency and availability reviewed.
- Payments, emails and external integrations checked where relevant.
- Monitoring observed during the post-deployment period.
- Rollback criteria documented and followed if needed.

## Production hygiene

- Debug/verbose modes disabled in production where applicable.
- Custom error pages provided for common errors (for example 404 and 500).
- Demo markers, fake badges and placeholder content removed.
- Favicon and custom domain configured where applicable.

## References

- Project deployment workflow: `docs/workflow-deploiement.md`

## Notes

- This checklist is a generic sequence and should be adapted to the project’s risk level and architecture.
