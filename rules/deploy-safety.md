# Deploy Safety

## Rollback-First Mentality

Every deployment must have a rollback plan before it starts. If you can't describe how to undo a deployment, you're not ready to deploy.

## Pre-Deploy Checklist

Before any deployment:

- [ ] Tests pass (unit, integration, E2E)
- [ ] Database migrations reviewed (if applicable)
- [ ] Rollback plan documented
- [ ] Deploy window is appropriate (not Friday, not end of day)
- [ ] Monitoring is in place

## Rules

- Never deploy without a verification plan
- Never deploy database migrations without testing rollback
- Never deploy on Fridays unless it's an emergency fix
- Always use `--dry-run` first for infrastructure changes
- Always have a rollback plan before deploying

## Post-Deploy

- Run health checks immediately after deploy
- Monitor for 24 hours
- Keep rollback plan accessible for the monitoring window
