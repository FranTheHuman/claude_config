# /minio:migrate — Evaluate and Plan MinIO Migration

Evaluate whether it's time to migrate MinIO out of the VPS, choose the right
migration target, and execute a zero-downtime migration.
Apply knowledge from `.claude/skills/minio-ops/SKILL.md`.

## Instructions

1. Read `.claude/skills/minio-ops/SKILL.md` section 10 (Migration Planning).
2. Ask for current metrics if not provided:
   - Current storage size: `mc du local/shalapp/`
   - Memory usage: `docker stats shalapp-minio --no-stream`
   - Disk usage: `df -h /var/lib/docker/volumes/`
3. Check migration signals against the thresholds in the skill.
4. Recommend migration target based on budget and scale.
5. Output the zero-downtime migration procedure.

## Migration Signal Thresholds

| Signal | Safe | Warning | Migrate Now |
|--------|------|---------|-------------|
| Memory | < 350M | 350-450M | > 450M sustained |
| Disk usage | < 50% | 50-70% | > 70% |
| Storage size | < 20GB | 20-50GB | > 50GB |
| Latency p95 | < 200ms | 200-500ms | > 500ms |
| OOM kills/week | 0 | 1 | 2+ |

## Migration Target Decision

| Budget | Volume | Recommendation |
|--------|--------|---------------|
| < $10/month | < 50GB | Stay on VPS, add disk |
| < $10/month | 50-200GB | Backblaze B2 ($1.20/TB) |
| < $20/month | Any | Cloudflare R2 (zero egress) |
| No limit | Any | AWS S3 or dedicated MinIO VPS |

## Output Structure

### Migration Assessment

**Current metrics**: [summary of provided metrics]
**Recommendation**: Stay / Evaluate / Migrate now
**Reason**: Why based on thresholds

### Target Recommendation

**Recommended target**: [B2 / R2 / S3 / other]
**Monthly cost estimate**: $X
**Why this target**: [cost, compatibility, migration ease]

### Zero-Downtime Migration Procedure

```bash
# Phase 1: Set up target
mc alias set target https://TARGET_ENDPOINT ACCESS_KEY SECRET_KEY

# Phase 2: Initial mirror (can take time for large buckets)
mc mirror --preserve local/shalapp target/your-target-bucket/

# Phase 3: Verify objects arrived correctly
mc diff local/shalapp/ target/your-target-bucket/

# Phase 4: Update API environment (in .env)
# S3_ENDPOINT=https://TARGET_ENDPOINT
# S3_ACCESS_KEY=new-key
# S3_SECRET_KEY=new-secret
# S3_BUCKET=your-target-bucket

# Phase 5: Restart only the API (MinIO still running as fallback)
docker compose restart shalapp-api

# Phase 6: Verify API works with new storage
curl https://yourapp.com/api/health
# Test an upload and download through the API

# Phase 7: Keep MinIO running for 48h as fallback
# Set up continuous sync during transition
mc mirror --watch --preserve local/shalapp target/your-target-bucket/ &

# Phase 8: After 48h, remove MinIO from compose
# Comment out minio and minio-init services
# docker compose up -d
```

### Post-migration cleanup

```bash
# After confirming migration is stable (1 week):

# 1. Stop MinIO
docker compose stop minio

# 2. Back up the volume one last time
mc mirror local/shalapp /opt/backups/minio/final/

# 3. Remove MinIO from compose file (comment out or delete)

# 4. Remove volume (IRREVERSIBLE - only after final backup confirmed)
# docker volume rm shalapp_minio_data
```

## Arguments

Usage: `/minio:migrate [evaluate|plan|execute|target]`

Examples:
- `/minio:migrate` → full assessment + recommendation + plan
- `/minio:migrate evaluate` → only the assessment against thresholds
- `/minio:migrate target` → only target comparison table
- `/minio:migrate execute b2` → migration procedure for Backblaze B2
- `/minio:migrate execute r2` → migration procedure for Cloudflare R2
- `/minio:migrate execute s3` → migration procedure for AWS S3
