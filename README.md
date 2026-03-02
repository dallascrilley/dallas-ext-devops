# dallas-ext-devops

Deployment, infrastructure, and file transfer tooling for Claude Code.

## Prerequisites

```bash
# rclone for S3/R2/B2 file transfers
brew install rclone  # or your package manager

# gh CLI for GitHub operations
brew install gh
```

## Contents

- **commands/deploy-docs** — GitHub Pages / static site deployment
- **commands/deploy-verify** — Post-deploy health checks and smoke tests
- **skills/rclone** — S3/R2/B2 upload patterns and setup validation
- **agents/deployment-verification** — Go/No-Go checklist and rollback planning
- **rules/deploy-safety** — Pre-deploy checklist, rollback-first mentality

## Installation

```bash
claude plugins install ~/Code/dallas-plugin-marketplace/dallas-ext-devops
```
