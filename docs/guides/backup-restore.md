# Qdrant Backup and Restore Guide

Quick reference for backing up and restoring Qdrant collections using the included scripts.

## Quick Start

### Create a Backup

```bash
./scripts/qdrant-backup.sh
```

This creates snapshots of **all collections** and saves them to:

```text
~/.arcaneum/backups/qdrant-snapshots-YYYY-MM-DD-HHMMSS/
```

### Restore from Backup

```bash
./scripts/qdrant-restore.sh ~/.arcaneum/backups/qdrant-snapshots-YYYY-MM-DD-HHMMSS/
```

## Backup Details

### What Gets Backed Up

The backup script:

- Connects to Qdrant via REST API
- Lists all collections
- Creates a snapshot for each collection using Qdrant's native snapshot feature
- Copies snapshots from the Docker container to local disk
- Generates a manifest file with backup metadata

### Backup Location

Default: `~/.arcaneum/backups/qdrant-snapshots-<TIMESTAMP>/`

Each backup directory contains:

```text
qdrant-snapshots-2025-12-29-143022/
├── manifest.txt                    # Backup metadata
├── pearldiver-...-.snapshot        # Collection snapshot
├── mycode-...-.snapshot            # Collection snapshot
└── ...
```

### Environment Variables

Customize backup behavior:

```bash
# Use different Qdrant host
QDRANT_HOST=http://localhost:7333 ./scripts/qdrant-backup.sh

# Change backup directory
BACKUP_DIR=/path/to/backups ./scripts/qdrant-backup.sh

# Use different container name
CONTAINER_NAME=my-qdrant ./scripts/qdrant-backup.sh
```

## Restore Details

### How Restore Works

The restore script:

1. Verifies backup directory exists
2. Finds all `.snapshot` files
3. Waits for Qdrant to be ready (health check)
4. For each snapshot:
   - Copies snapshot into container's `/qdrant/snapshots/` directory
   - Triggers restore via Qdrant's recovery API
   - Verifies point and vector counts

### Requirements

- Qdrant must be running (`arc container start`)
- Backup directory must contain `.snapshot` files
- Sufficient disk space in Docker volumes

### Environment Variables

```bash
# Restore to different Qdrant instance
QDRANT_HOST=http://localhost:7333 ./scripts/qdrant-restore.sh /path/to/backup/

# Use different container name
CONTAINER_NAME=my-qdrant ./scripts/qdrant-restore.sh /path/to/backup/
```

## Common Workflows

### Regular Backups

Create a backup before risky operations:

```bash
# Before major updates
./scripts/qdrant-backup.sh

# Before deleting collections
./scripts/qdrant-backup.sh

# Before system upgrades
./scripts/qdrant-backup.sh
```

### Automated Backups

Add to crontab for daily backups:

```bash
# Daily at 2 AM
0 2 * * * /path/to/arcaneum/scripts/qdrant-backup.sh
```

### Disaster Recovery

```bash
# 1. Stop Qdrant
arc container stop

# 2. Remove corrupted data (optional)
docker volume rm qdrant-arcaneum-storage

# 3. Start fresh instance
arc container start

# 4. Restore from backup
./scripts/qdrant-restore.sh ~/.arcaneum/backups/qdrant-snapshots-YYYY-MM-DD-HHMMSS/

# 5. Verify collections
curl http://localhost:6333/collections
```

### Backup Single Collection

The scripts back up all collections. For a single collection:

```bash
# Create snapshot
curl -X POST "http://localhost:6333/collections/pearldiver/snapshots"

# Copy from container
docker cp arcaneum-qdrant:/qdrant/snapshots/pearldiver/<snapshot-file> ./pearldiver-backup.snapshot
```

### Cloud Backup

Upload backups to cloud storage:

```bash
# Create backup
./scripts/qdrant-backup.sh

# Get latest backup directory
LATEST=$(ls -td ~/.arcaneum/backups/qdrant-snapshots-* | head -1)

# Upload with rclone (Google Drive, S3, etc.)
rclone copy "$LATEST" remote:arcaneum-backups/

# Or tar and upload
tar czf qdrant-backup.tar.gz -C ~/.arcaneum/backups "$(basename $LATEST)"
# Then upload qdrant-backup.tar.gz to your cloud provider
```

## Verification

### Check Backup Success

```bash
# View backup manifest
cat ~/.arcaneum/backups/qdrant-snapshots-YYYY-MM-DD-HHMMSS/manifest.txt

# List snapshot files
ls -lh ~/.arcaneum/backups/qdrant-snapshots-YYYY-MM-DD-HHMMSS/*.snapshot

# Check snapshot sizes (should not be empty)
du -sh ~/.arcaneum/backups/qdrant-snapshots-YYYY-MM-DD-HHMMSS/*.snapshot
```

### Check Restore Success

```bash
# List all collections
curl http://localhost:6333/collections | python3 -m json.tool

# Check specific collection
curl http://localhost:6333/collections/pearldiver | python3 -m json.tool

# Verify via CLI
arc collection list
arc collection info pearldiver
```

## Troubleshooting

### Backup Fails

**Problem:** "No collections found"

```bash
# Check Qdrant is running
curl http://localhost:6333/healthz

# Check collections exist
curl http://localhost:6333/collections
```

**Problem:** "Failed to create snapshot"

- Check disk space: `df -h`
- Check container logs: `docker logs qdrant-arcaneum`
- Verify container name: `docker ps`

### Restore Fails

**Problem:** "Backup directory not found"

- Verify path exists: `ls -la /path/to/backup/`
- Use absolute path, not relative

**Problem:** "No snapshot files found"

- Check for `.snapshot` files: `ls *.snapshot`
- Verify backup completed successfully

**Problem:** "Qdrant not responding"

- Start Qdrant: `arc container start`
- Wait for readiness: `curl http://localhost:6333/healthz`
- Check container status: `docker ps`

**Problem:** "Collection info shows 0 points"

- Check restore logs for errors
- Verify snapshot file not corrupted: `file backup.snapshot`
- Try restoring again
- Check container logs: `docker logs qdrant-arcaneum`

## Best Practices

1. **Regular Backups**: Create backups before major changes
2. **Test Restores**: Periodically test restore process
3. **Multiple Locations**: Store backups in multiple locations (local + cloud)
4. **Retention Policy**: Keep last N backups, delete older ones
5. **Document Backups**: Note what each backup contains
6. **Verify After Restore**: Always check collection counts after restore

## See Also

- [Qdrant Migration Guide](./qdrant-migration.md) - Migrating from bind mounts to named volumes
- [Qdrant Snapshots API](https://qdrant.tech/documentation/concepts/snapshots/) - Official documentation
- [CLI Reference](./cli-reference.md) - Arc CLI commands
