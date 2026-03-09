---
name: hetzner-cloud
description: "Use when the user needs to manage Hetzner Cloud infrastructure — servers, volumes, firewalls, networks, snapshots, or SSH keys — via the hcloud CLI."
---

# Hetzner Cloud CLI

Manage Hetzner Cloud infrastructure via the `hcloud` CLI.

## Safety Rules

- **NEVER execute delete commands.** All destructive operations are forbidden.
- **NEVER expose or log API tokens.**
- **ALWAYS show the exact command and wait for explicit approval** before create/modify operations.
- **ALWAYS suggest a snapshot** before any modification:
  ```bash
  hcloud server create-image <server> --type snapshot --description "Backup before changes"
  ```

## Setup

```bash
# Check existing contexts
hcloud context list

# Create new context (prompts for API token)
hcloud context create <context-name>

# Switch context
hcloud context use <context-name>
```

Get API token: https://console.hetzner.cloud/ → Project → Security → API Tokens → Generate (read+write).

## Installation

```bash
# macOS
brew install hcloud

# Debian/Ubuntu
sudo apt update && sudo apt install hcloud-cli

# Fedora
sudo dnf install hcloud
```

## Commands

### Servers
```bash
hcloud server list
hcloud server describe <name>
hcloud server create --name my-server --type cx22 --image ubuntu-24.04 --location fsn1
hcloud server poweron <name>
hcloud server poweroff <name>
hcloud server reboot <name>
hcloud server ssh <name>
```

### Discovery
```bash
hcloud server-type list
hcloud location list
hcloud datacenter list
```

### Firewalls
```bash
hcloud firewall create --name my-firewall
hcloud firewall add-rule <name> --direction in --protocol tcp --port 22 --source-ips 0.0.0.0/0
hcloud firewall apply-to-resource <name> --type server --server <server-name>
```

### Networks
```bash
hcloud network create --name my-network --ip-range 10.0.0.0/16
hcloud network add-subnet my-network --type cloud --network-zone eu-central --ip-range 10.0.0.0/24
hcloud server attach-to-network <server> --network <network>
```

### Volumes
```bash
hcloud volume create --name my-volume --size 100 --location fsn1
hcloud volume attach <volume> --server <server>
hcloud volume detach <volume>
```

### Snapshots
```bash
hcloud server create-image <server> --type snapshot --description "My snapshot"
hcloud image list --type snapshot
```

### SSH Keys
```bash
hcloud ssh-key list
hcloud ssh-key create --name my-key --public-key-from-file ~/.ssh/id_rsa.pub
```

## Output Formats

```bash
hcloud server list -o json
hcloud server list -o yaml
hcloud server list -o columns=id,name,status
```

## Tips

- Use `--selector` with labels for bulk operations across multiple resources
- Use contexts to manage multiple Hetzner projects from one machine
