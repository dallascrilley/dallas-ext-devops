---
name: rclone
description: S3/R2/B2 file transfer patterns using rclone CLI. Setup validation, common operations, and troubleshooting.
---

# Rclone File Transfer

## Setup Validation

Before any operation, verify rclone is configured:

```bash
# Check rclone is installed
rclone version

# List configured remotes
rclone listremotes

# Test connectivity to a remote
rclone lsd <remote>:

# Check specific bucket
rclone ls <remote>:<bucket> --max-depth 1
```

## Common Operations

### Copy (non-destructive)
```bash
# Copy local files to remote
rclone copy ./dist <remote>:<bucket>/path --progress

# Copy with dry run first
rclone copy ./dist <remote>:<bucket>/path --dry-run
rclone copy ./dist <remote>:<bucket>/path --progress
```

### Sync (mirror — deletes extras on destination)
```bash
# Always dry-run first
rclone sync ./dist <remote>:<bucket>/path --dry-run

# Then sync
rclone sync ./dist <remote>:<bucket>/path --progress
```

### Mount (FUSE filesystem)
```bash
# Mount remote as local directory
rclone mount <remote>:<bucket> /mnt/remote --daemon

# Unmount
fusermount -u /mnt/remote
```

## Provider-Specific Notes

| Provider | Remote Type | Notes |
|----------|------------|-------|
| AWS S3 | `s3` | Standard S3 API |
| Cloudflare R2 | `s3` | S3-compatible, set endpoint |
| Backblaze B2 | `b2` | Native B2 API or S3-compatible |

## Safety

- Always `--dry-run` before `sync` (sync deletes files not in source)
- Use `copy` instead of `sync` when you don't want to delete destination files
- Add `--backup-dir <remote>:<bucket>/backup` to sync for safety
- Never sync to a bucket that contains data you haven't backed up
