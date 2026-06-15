# /minio:health — Check MinIO Health and Resource Usage

Run a complete health check of the MinIO instance running on this VPS.
Check container status, API health endpoint, memory usage, disk usage, and logs.
Apply knowledge from `.claude/skills/minio-ops/SKILL.md`.

## Instructions

1. Read `.claude/skills/minio-ops/SKILL.md` section 2 (Daily Operations) before running.
2. Run all diagnostic commands in sequence.
3. Interpret results and flag any issue with severity.
4. If any threshold is exceeded, provide the specific action to take.

## Diagnostic sequence

```bash
# 1. Container status
docker inspect shalapp-minio --format '{{.State.Status}} | health: {{.State.Health.Status}}'

# 2. API health endpoint
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:9000/minio/health/live

# 3. Memory usage
docker stats shalapp-minio --no-stream --format "{{.MemUsage}} / {{.MemPerc}}"

# 4. Disk usage (MinIO volume)
docker system df -v | grep minio_data
df -h /var/lib/docker/volumes/

# 5. Recent errors in logs
docker logs shalapp-minio --since 1h 2>&1 | grep -i "error\|warn\|fail\|OOM" | tail -20

# 6. VPS overall resources
free -h && df -h /
```

## Output Structure

### MinIO Status
- Container: ✅ running / ⚠️ unhealthy / ❌ stopped
- API: ✅ 200 OK / ❌ not responding
- Memory: X MB / 512 MB limit (X%)
- Disk: X GB used / X GB available

### Flags (if any)
```
[CRITICAL] Memory > 450MB — approaching 512MB limit
[CRITICAL] API not responding — check logs immediately
[CRITICAL] Disk > 70% — evaluate migration
[WARNING]  Memory > 350MB consistently — monitor closely
[WARNING]  Errors in last hour — review log output
```

### Recommended Action
Specific next step based on findings.

## Arguments

Usage: `/minio:health` — no arguments needed, reads current deployment state.
