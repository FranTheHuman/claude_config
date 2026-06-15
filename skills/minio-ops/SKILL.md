---
name: minio-ops
description: >
  Expert operational knowledge for running MinIO on a small VPS in Docker Compose.
  Covers day-to-day operations, health monitoring, bucket and policy management,
  TLS configuration, performance tuning for constrained environments, backup strategies,
  access control, S3 SDK integration (Java/Node.js), troubleshooting, migration planning,
  and the decision criteria for when to move MinIO out of the VPS. Designed for a
  single-node MinIO instance serving a production Node.js API with PostgreSQL and Redis
  on the same host.
---

You are an expert MinIO operator for small VPS deployments. Apply this skill as the
authoritative source for every MinIO decision. The guiding principle:
**keep it running reliably on constrained resources, and know exactly when it's time to move it out**.

---

## 1. Current Deployment Context

The deployment uses Docker Compose with the following MinIO setup:

```yaml
# Key facts about the current setup:
image: quay.io/minio/minio:latest          # standard MinIO (not AIStor)
container: shalapp-minio
ports:
  - "127.0.0.1:9000:9000"   # S3 API — loopback only
  - "127.0.0.1:9002:9001"   # Console — loopback only (SSH tunnel to access)
memory limit: 512M
data volume: minio_data
network: shalapp-net (internal Docker network)
bucket: configured via minio-init container
public paths: logo/, blog/ (anonymous download)
private paths: everything else (auth required)

# Co-tenants on same VPS:
# - shalapp-api (Node.js, 1G limit)
# - shalapp-db (PostgreSQL 16, 1G limit)
# - shalapp-redis (Redis, 256M limit)
# - shalapp-web (frontend, 128M)
# - shalapp-backoffice (frontend, 128M)
# - portainer (128M)
# Total reserved: ~3.2G RAM on one VPS
```

**Access pattern**: MinIO is accessed only by `shalapp-api` over the internal Docker
network (`http://minio:9000`). No direct external access. Console accessed via SSH tunnel.

---

## 2. Daily Operations

### Check MinIO health

```bash
# Health check — returns 200 OK if healthy
curl -s http://127.0.0.1:9000/minio/health/live && echo "✅ MinIO healthy"

# Full status via mc (if mc is installed on host)
mc alias set local http://127.0.0.1:9000 $MINIO_ROOT_USER $MINIO_ROOT_PASSWORD
mc admin info local

# Check container status and resource usage
docker stats shalapp-minio --no-stream
docker inspect shalapp-minio --format '{{.State.Health.Status}}'
```

### Check logs

```bash
# Last 50 lines
docker logs shalapp-minio --tail 50

# Follow in real time
docker logs shalapp-minio -f

# Last hour only
docker logs shalapp-minio --since 1h

# Look for errors only
docker logs shalapp-minio 2>&1 | grep -i "error\|warn\|fail"
```

### Restart MinIO safely

```bash
# Graceful restart (preferred)
docker compose restart minio

# If compose file is not in current directory
docker compose -f /path/to/docker-compose.yml restart minio

# Hard restart (if graceful fails)
docker stop shalapp-minio && docker start shalapp-minio
```

### Access the console via SSH tunnel

```bash
# From local machine:
ssh -L 9002:127.0.0.1:9002 user@your-vps-ip -N &
# Then open: http://localhost:9002 in browser
```

---

## 3. Bucket and Object Management

### List buckets and objects

```bash
mc ls local/                          # list buckets
mc ls local/shalapp/                  # list objects in bucket
mc ls local/shalapp/blog/             # list blog objects
mc ls --recursive local/shalapp/      # recursive listing
mc du local/shalapp/                  # disk usage
```

### Upload and download

```bash
mc cp ./logo.png local/shalapp/logo/logo.png
mc cp local/shalapp/logo/logo.png ./logo-backup.png
mc rm local/shalapp/blog/old-post.md
```

### Check bucket policy

```bash
mc anonymous get local/shalapp/logo    # should show "download"
mc anonymous get local/shalapp/blog    # should show "download"
mc anonymous get local/shalapp/        # should show "none" (private)
```

### Reset public policy (if lost after restart)

```bash
mc anonymous set download local/shalapp/logo
mc anonymous set download local/shalapp/blog
```

### Create a pre-signed URL (for private objects)

```bash
# URL valid for 24 hours
mc share download --expire 24h local/shalapp/private/document.pdf
```

---

## 4. Access Control and Users

### Create a service account (for the API)

```bash
# Create user with limited permissions
mc admin user add local api-user strong-password-here

# Attach a policy (use built-in or custom)
mc admin policy attach local readwrite --user api-user
```

### Create a custom policy (recommended for production)

```json
// api-policy.json — allow only the shalapp bucket
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::shalapp",
        "arn:aws:s3:::shalapp/*"
      ]
    }
  ]
}
```

```bash
mc admin policy create local api-policy api-policy.json
mc admin policy attach local api-policy --user api-user
```

### Rotate credentials without downtime

```bash
# 1. Create new service account
mc admin user add local api-user-v2 new-password

# 2. Attach same policy
mc admin policy attach local api-policy --user api-user-v2

# 3. Update .env in the API service → MINIO_ROOT_USER / S3_ACCESS_KEY

# 4. Restart API (not MinIO)
docker compose restart shalapp-api

# 5. Remove old user after confirming API works
mc admin user remove local api-user
```

---

## 5. Node.js SDK Integration (for the shalapp-api)

### Installation

```bash
npm install @aws-sdk/client-s3 @aws-sdk/s3-request-presigner
```

### Client configuration (pointing to internal Docker network)

```typescript
// lib/storage.ts
import { S3Client, PutObjectCommand, GetObjectCommand, DeleteObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

// When running inside Docker Compose — use service name "minio"
// When running locally — use 127.0.0.1:9000
const endpoint = process.env.S3_ENDPOINT ?? 'http://minio:9000';

export const s3 = new S3Client({
    endpoint,
    region: 'us-east-1',              // MinIO ignores region but SDK requires it
    credentials: {
        accessKeyId: process.env.S3_ACCESS_KEY!,
        secretAccessKey: process.env.S3_SECRET_KEY!,
    },
    forcePathStyle: true,             // REQUIRED for MinIO — never use virtual-hosted style
});

export const BUCKET = process.env.S3_BUCKET ?? 'shalapp';

// Upload a file
export async function uploadFile(key: string, body: Buffer, contentType: string) {
    await s3.send(new PutObjectCommand({
        Bucket: BUCKET,
        Key: key,
        Body: body,
        ContentType: contentType,
    }));
    // Return public URL for logo/ and blog/ paths
    return `${process.env.MINIO_PUBLIC_URL}/${BUCKET}/${key}`;
}

// Generate a pre-signed URL for private objects (expires in 1 hour)
export async function getPresignedUrl(key: string, expiresIn = 3600) {
    return getSignedUrl(s3, new GetObjectCommand({ Bucket: BUCKET, Key: key }), { expiresIn });
}

// Delete an object
export async function deleteFile(key: string) {
    await s3.send(new DeleteObjectCommand({ Bucket: BUCKET, Key: key }));
}
```

### Critical configuration rule

```typescript
// ALWAYS set forcePathStyle: true when connecting to MinIO
// Without it, the SDK tries to reach <bucket>.minio:9000 which doesn't resolve

// CORRECT
new S3Client({ endpoint: 'http://minio:9000', forcePathStyle: true })

// WRONG — bucket name prepended to hostname, DNS fails
new S3Client({ endpoint: 'http://minio:9000', forcePathStyle: false })
```

---

## 6. Performance Tuning for Constrained VPS

### Memory limits — current setup is correct

```yaml
# 512M for MinIO is adequate for:
# - Up to ~50 concurrent connections
# - Objects up to several GB
# - Bucket listing of up to ~100k objects
# Increase to 768M if you see OOM kills:
# docker logs shalapp-minio 2>&1 | grep -i "OOM\|killed"
```

### Disk I/O — the real bottleneck on VPS

```bash
# Check disk usage
df -h /var/lib/docker/volumes/

# Check I/O wait (high iowait = disk is the bottleneck)
iostat -x 2 5

# Check volume size
docker system df -v | grep minio_data
```

### What to do when disk fills up

```bash
# 1. Check what's using space
mc du --versions local/shalapp/

# 2. List largest objects
mc ls --recursive local/shalapp/ | sort -k5 -rn | head -20

# 3. Remove objects you don't need
mc rm local/shalapp/old-uploads/

# 4. Compact if using versioning (removes delete markers)
# mc rm --versions --recursive local/shalapp/path/

# 5. Check Docker overhead
docker system prune -f   # removes unused images/containers (NOT volumes)
```

### Logging — current setup is correct

```yaml
# Current log rotation is correct:
logging:
  driver: "json-file"
  options:
    max-size: "10m"   # rotate at 10MB
    max-file: "3"     # keep 3 rotated files = max 30MB logs
```

---

## 7. Backup Strategy

### Full backup of MinIO data (run on VPS)

```bash
#!/bin/bash
# /opt/scripts/backup-minio.sh
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/opt/backups/minio"
mkdir -p "$BACKUP_DIR"

# Mirror all MinIO data to local backup directory
mc mirror --overwrite local/shalapp "$BACKUP_DIR/$DATE/"

# Compress
tar -czf "$BACKUP_DIR/minio-$DATE.tar.gz" -C "$BACKUP_DIR" "$DATE/"
rm -rf "$BACKUP_DIR/$DATE/"

# Keep only last 7 backups
ls -t "$BACKUP_DIR"/*.tar.gz | tail -n +8 | xargs rm -f

echo "✅ MinIO backup complete: minio-$DATE.tar.gz"
```

### Offsite backup (sync to external S3)

```bash
# Sync to Backblaze B2 or AWS S3 for offsite copy
mc mirror local/shalapp b2/your-backup-bucket/minio/
```

### Restore from backup

```bash
# 1. Ensure MinIO is running
# 2. Extract backup
tar -xzf minio-20250101_120000.tar.gz

# 3. Restore objects
mc mirror --overwrite ./20250101_120000/ local/shalapp/

# 4. Re-apply public policies
mc anonymous set download local/shalapp/logo
mc anonymous set download local/shalapp/blog
```

---

## 8. TLS Configuration

### Current setup: no TLS (correct for VPS with internal-only access)

The current setup is correct because:
- MinIO ports are bound to `127.0.0.1` only (loopback)
- API access is via internal Docker network (no public exposure)
- Console access is via SSH tunnel (encrypted at SSH layer)

### Add TLS when you need external direct access (optional)

```bash
# Generate self-signed cert for MinIO
mkdir -p /opt/minio/certs
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /opt/minio/certs/private.key \
    -out /opt/minio/certs/public.crt \
    -subj "/CN=your-vps-ip"

# Mount in compose:
volumes:
  - /opt/minio/certs:/root/.minio/certs
```

### Add TLS when using a reverse proxy (Nginx)

```nginx
# Nginx proxies HTTPS → MinIO internally (no MinIO TLS needed)
server {
    listen 443 ssl;
    server_name storage.yourapp.com;

    ssl_certificate /etc/letsencrypt/live/yourapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourapp.com/privkey.pem;

    client_max_body_size 0;       # required for large uploads
    proxy_buffering off;           # required for streaming
    proxy_request_buffering off;   # required for streaming

    location / {
        proxy_pass http://127.0.0.1:9000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

---

## 9. Troubleshooting

### MinIO won't start

```bash
# Check logs
docker logs shalapp-minio --tail 100

# Common causes:
# 1. Volume permission issue
docker exec shalapp-minio ls -la /data

# 2. Port conflict
ss -tlnp | grep 9000

# 3. Memory OOM kill
dmesg | grep -i "killed process" | grep minio

# 4. Corrupt data directory (rare)
# Solution: restore from backup
```

### minio-init fails (bucket not created)

```bash
# Check init container logs
docker logs shalapp-minio-init

# Manual bucket creation
mc alias set local http://127.0.0.1:9000 $MINIO_ROOT_USER $MINIO_ROOT_PASSWORD
mc mb --ignore-existing local/shalapp
mc anonymous set download local/shalapp/logo
mc anonymous set download local/shalapp/blog
```

### API can't connect to MinIO

```bash
# 1. Check if MinIO is healthy from inside the API container
docker exec shalapp-api curl -s http://minio:9000/minio/health/live

# 2. Verify same network
docker network inspect shalapp-net | grep -A3 "shalapp-minio\|shalapp-api"

# 3. Check env vars in API
docker exec shalapp-api env | grep -i "minio\|s3"

# 4. Common fix: forcePathStyle not set to true in SDK
```

### Uploads fail with "413 Request Entity Too Large"

```bash
# If using Nginx in front:
# Add to nginx config:
client_max_body_size 0;
proxy_request_buffering off;
```

### Objects not publicly accessible

```bash
# Verify policy
mc anonymous get local/shalapp/logo

# Re-apply if needed
mc anonymous set download local/shalapp/logo
mc anonymous set download local/shalapp/blog
```

### High memory usage

```bash
# Check current usage
docker stats shalapp-minio --no-stream --format "{{.MemUsage}}"

# If consistently > 400M, increase limit in compose:
# memory: 768M

# If getting OOM killed:
# dmesg | grep minio
```

---

## 10. Migration Planning — When to Move MinIO Out

### Signals that it's time to migrate

| Signal | Threshold | Action |
|--------|-----------|--------|
| MinIO memory consistently at limit | > 450M for 7+ days | Increase VPS RAM or migrate |
| Disk usage | > 70% of VPS disk | Add disk or migrate |
| Upload/download latency | > 500ms p95 | Migrate to dedicated storage |
| Storage size | > 50GB | Consider Backblaze B2 or AWS S3 |
| VPS CPU steal time | > 10% during uploads | VPS is overloaded |
| More than 2 OOM kills per week | Any | Migrate immediately |

### Migration targets (in order of ease)

1. **Backblaze B2** — S3-compatible, cheapest egress, easy migration
2. **Cloudflare R2** — Zero egress costs, S3-compatible
3. **AWS S3** — Most compatible, most expensive
4. **DigitalOcean Spaces** — S3-compatible, simple pricing
5. **Dedicated VPS with MinIO** — Same setup, more resources

### Migration procedure (zero-downtime)

```bash
# Step 1: Set up target (e.g., Backblaze B2)
mc alias set b2 https://s3.us-west-004.backblazeb2.com KEY_ID APPLICATION_KEY

# Step 2: Mirror all objects
mc mirror --preserve local/shalapp b2/your-b2-bucket

# Step 3: Update API environment variables
# S3_ENDPOINT=https://s3.us-west-004.backblazeb2.com
# S3_ACCESS_KEY=new-key
# S3_SECRET_KEY=new-secret
# S3_BUCKET=your-b2-bucket

# Step 4: Restart API only (MinIO keeps running as fallback)
docker compose restart shalapp-api

# Step 5: Verify API works with new storage

# Step 6: Set up sync to keep both in sync during transition
mc mirror --watch local/shalapp b2/your-b2-bucket &

# Step 7: After 48h verification, remove MinIO from compose
```

---

## 11. Compose File Management

### Update MinIO image

```bash
# Pull latest image
docker compose pull minio

# Recreate container with new image (brief downtime ~5s)
docker compose up -d --no-deps minio

# Verify after update
curl -s http://127.0.0.1:9000/minio/health/live
```

### Scale down for VPS maintenance

```bash
# Stop MinIO temporarily (data persists in volume)
docker compose stop minio

# Start again
docker compose start minio

# Check minio-init doesn't re-run (it only runs once due to depends_on completed)
```

### Environment variables that must be in .env

```bash
MINIO_ROOT_USER=       # admin username — keep strong
MINIO_ROOT_PASSWORD=   # admin password — min 8 chars
S3_BUCKET=shalapp      # bucket name
```

---

## 12. Architecture Decision Guide

| Question | Answer |
|----------|--------|
| Should MinIO be on the same VPS as the API? | Yes, for current scale — saves egress costs and latency |
| Should MinIO ports be exposed publicly? | No — loopback only, SSH tunnel for console |
| Should I enable versioning? | Only if you need file history — adds storage overhead |
| Should I enable SSE encryption? | Not required for VPS; add if compliance required |
| Should I use the root user in the API? | No — create a limited service account |
| When to add TLS to MinIO directly? | Only if direct external access needed without reverse proxy |
| Should I run multiple MinIO instances? | No — distributed mode requires ≥4 nodes, defeats purpose on small VPS |
| Should I upgrade to MinIO AIStor? | Only if you need enterprise features — MinIO OSS is sufficient here |

---

## Reference

- MinIO mc client: https://min.io/docs/minio/linux/reference/minio-mc.html
- MinIO S3 compatibility: https://min.io/docs/minio/linux/reference/s3-api-compatibility.html
- AWS SDK v3 for Node.js: https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/clients/client-s3/
- Backblaze B2 migration: https://help.backblaze.com/hc/en-us/articles/360048671111
- Cloudflare R2: https://developers.cloudflare.com/r2/
