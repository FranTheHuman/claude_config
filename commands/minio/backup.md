# /minio:backup — Backup MinIO Data

Generate the backup script and commands for the current MinIO deployment.
Covers local backup, offsite sync, and restore procedure.
Apply knowledge from `.claude/skills/minio-ops/SKILL.md`.

## Instructions

1. Read `.claude/skills/minio-ops/SKILL.md` section 7 (Backup Strategy).
2. Provide the complete backup script ready to copy to the VPS.
3. Include the cron job to automate it.
4. Include the restore procedure.
5. Always include the policy restore step after data restore.

## Output Structure

### Local backup script

```bash
#!/bin/bash
# /opt/scripts/backup-minio.sh
# Usage: bash backup-minio.sh
# Cron: 0 2 * * * /opt/scripts/backup-minio.sh >> /var/log/minio-backup.log 2>&1

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/opt/backups/minio"
BUCKET="${S3_BUCKET:-shalapp}"

# Load env if needed
source /opt/app/.env 2>/dev/null || true

mkdir -p "$BACKUP_DIR"

echo "[$DATE] Starting MinIO backup..."

# Set up mc alias
mc alias set local http://127.0.0.1:9000 "$MINIO_ROOT_USER" "$MINIO_ROOT_PASSWORD" \
    --api S3v4 --path auto 2>/dev/null

# Mirror bucket to local directory
mc mirror --overwrite "local/$BUCKET" "$BACKUP_DIR/latest/"

# Compress snapshot
tar -czf "$BACKUP_DIR/minio-$DATE.tar.gz" -C "$BACKUP_DIR" "latest/"

# Keep only last 7 daily backups
ls -t "$BACKUP_DIR"/minio-*.tar.gz | tail -n +8 | xargs rm -f 2>/dev/null

echo "[$DATE] ✅ Backup complete: minio-$DATE.tar.gz"
echo "[$DATE] Size: $(du -sh "$BACKUP_DIR/minio-$DATE.tar.gz" | cut -f1)"
```

### Install backup script

```bash
mkdir -p /opt/scripts /opt/backups/minio
# paste script above
chmod +x /opt/scripts/backup-minio.sh

# Add to cron (runs at 2am daily)
(crontab -l 2>/dev/null; echo "0 2 * * * /opt/scripts/backup-minio.sh >> /var/log/minio-backup.log 2>&1") | crontab -

# Test run
bash /opt/scripts/backup-minio.sh
```

### Offsite sync (recommended)

```bash
# Sync to Backblaze B2 (cheapest option)
mc alias set b2 https://s3.us-west-004.backblazeb2.com B2_KEY_ID B2_APP_KEY
mc mirror --overwrite local/shalapp b2/your-backup-bucket/

# Or to Cloudflare R2 (zero egress cost)
mc alias set r2 https://ACCOUNT_ID.r2.cloudflarestorage.com CF_ACCESS_KEY CF_SECRET_KEY
mc mirror --overwrite local/shalapp r2/your-r2-bucket/
```

### Restore procedure

```bash
# 1. Ensure MinIO is running and healthy
curl -s http://127.0.0.1:9000/minio/health/live

# 2. Extract backup
cd /opt/backups/minio
tar -xzf minio-20250101_020000.tar.gz

# 3. Restore objects
mc alias set local http://127.0.0.1:9000 $MINIO_ROOT_USER $MINIO_ROOT_PASSWORD
mc mirror --overwrite ./latest/ local/shalapp/

# 4. CRITICAL: re-apply public policies (mc mirror does NOT restore policies)
mc anonymous set download local/shalapp/logo
mc anonymous set download local/shalapp/blog

# 5. Verify
mc ls local/shalapp/
mc anonymous get local/shalapp/logo
```

## Arguments

Usage: `/minio:backup [local|offsite|restore|script]`

Examples:
- `/minio:backup` → shows full backup setup
- `/minio:backup script` → shows only the backup script
- `/minio:backup restore` → shows only the restore procedure
- `/minio:backup offsite` → shows offsite sync options (B2, R2, S3)
