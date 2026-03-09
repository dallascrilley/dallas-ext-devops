# dallas-ext-devops

> L1 extension plugin providing deployment, infrastructure, and file transfer tooling. Extends L0 safety rules from dallas-base-plugin.

## Structure

- `commands/` — deploy-docs and deploy-verify commands
- `skills/` — rclone file transfer skill, github-ci-setup infrastructure scaffolding, hetzner-cloud server/network/volume management
- `agents/` — deployment-verification agent
- `rules/` — deploy-safety conventions

## Prerequisites

- `rclone` CLI for file transfer operations
- `gh` CLI for GitHub Pages deployments
- `hcloud` CLI for Hetzner Cloud management (`brew install hcloud`)
