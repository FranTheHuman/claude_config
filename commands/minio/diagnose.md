# /minio:diagnose — Diagnose a MinIO Error or Problem

Analyze a specific MinIO error, connection failure, or unexpected behavior.
Identify the root cause, explain the impact, and provide the exact fix.
Apply knowledge from `.claude/skills/minio-ops/SKILL.md`.

## Instructions

1. Read `.claude/skills/minio-ops/SKILL.md` section 9 (Troubleshooting).
2. If no error message is provided, ask for:
   - The exact error message or HTTP status code
   - Output of: `docker logs shalapp-minio --tail 50`
   - Output of: `docker inspect shalapp-minio --format '{{.State.Health.Status}}'`
3. Classify root cause from the table below.
4. Output diagnosis in the structure below.

## Root Cause Categories

| Category | Typical Symptoms |
|----------|-----------------|
| **forcePathStyle missing** | SDK error: bucket name prepended to hostname, DNS resolution fails |
| **Wrong endpoint** | Timeout from API; correct endpoint inside Docker is `http://minio:9000` |
| **Different Docker network** | API cannot reach MinIO by service name; containers on different networks |
| **Policy lost after restart** | 403 on public logo/blog paths; `mc anonymous get` shows `none` |
| **minio-init failed** | Bucket doesn't exist; API gets NoSuchBucket error on startup |
| **OOM kill** | Container keeps restarting; `dmesg | grep minio` shows killed process |
| **413 upload error** | Large file upload fails; Nginx in front has default 1MB limit |
| **Disk full** | Write operations fail; 500 errors; `df -h` shows 100% usage |
| **Corrupt credentials in env** | 403 Access Denied on all operations; root user wrong |
| **Volume permissions** | MinIO can't write to /data; starts then immediately fails |

## Output Structure

### Diagnosis

**Root cause**: [category from table above]

**What happened**: Clear explanation of why this fails in the context of
Docker networking, MinIO's S3 path style, or resource constraints.

**Diagnostic commands to confirm**:
```bash
# Commands to run to confirm this is the actual root cause
```

### Fix

```bash
# Exact commands to resolve the issue
```

**Why this fixes it**: One sentence connecting the fix to the root cause.

### Prevention

The configuration change or monitoring habit to prevent recurrence.

## Arguments

Usage: `/minio:diagnose [error description or paste error directly]`

Examples:
- `/minio:diagnose` → agent asks for error details
- `/minio:diagnose NoSuchBucket when API starts`
- `/minio:diagnose 403 on public image URLs`
- `/minio:diagnose connection refused from API container`
- `/minio:diagnose upload fails with 413`
- `/minio:diagnose MinIO container keeps restarting`
- `/minio:diagnose SDK error: getaddrinfo ENOTFOUND shalapp.minio`
- `/minio:diagnose disk is at 85% on the VPS`
