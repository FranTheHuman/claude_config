# /nginx:diagnose — Diagnose an Nginx Error or Problem

Analyze a specific Nginx error, HTTP status code, or unexpected behavior.
Identify the root cause and provide the exact fix.
Apply knowledge from `.claude/skills/nginx-ops/SKILL.md`.

## Instructions

1. Read `.claude/skills/nginx-ops/SKILL.md` section 10 (Troubleshooting).
2. If no error provided, ask for:
   - The HTTP status code or error message
   - Output of: `sudo tail -20 /var/log/nginx/error.log`
   - The relevant config block
3. Classify root cause from the table below.
4. Output diagnosis in the structure below.

## Root Cause Categories

| Category | Symptoms |
|----------|---------|
| **Backend down** | 502 — backend container not running or crashed |
| **Wrong upstream port/address** | 502 — port mismatch in proxy_pass |
| **Docker network isolation** | 502 — Nginx can't reach backend by service name |
| **Body size limit** | 413 — file upload exceeds `client_max_body_size` |
| **Upstream timeout** | 504 — backend too slow, `proxy_read_timeout` too short |
| **MinIO streaming broken** | Upload/download stalls — missing `proxy_buffering off` |
| **TLS certificate expired** | 495/SSL error — cert needs renewal |
| **Rate limit hit** | 429 — legitimate traffic being throttled |
| **Location block conflict** | 403/404 — wrong `location` block matching order |
| **Config syntax error** | Nginx fails to start after reload |

## Output Structure

### Diagnosis

**Root cause**: [category from table above]

**What happened**: Clear explanation of why this fails — not just "port is wrong"
but why Nginx can't reach it given the Docker network and proxy setup.

**Diagnostic commands to confirm**:
```bash
# Commands to run to confirm this is the actual root cause
```

### Fix

```nginx
# Corrected Nginx config block
```

```bash
# Commands to apply the fix and verify
sudo nginx -t && sudo systemctl reload nginx
```

**Why this fixes it**: One sentence connecting fix to root cause.

### Prevention

The configuration rule or monitoring habit to prevent recurrence.

## Arguments

Usage: `/nginx:diagnose [error or description]`

Examples:
- `/nginx:diagnose` → agent asks for error details
- `/nginx:diagnose 502 on all API requests after deploying`
- `/nginx:diagnose 413 when uploading files larger than 10MB`
- `/nginx:diagnose 504 on slow database queries`
- `/nginx:diagnose MinIO uploads stall at 50%`
- `/nginx:diagnose Nginx won't reload after config change`
- `/nginx:diagnose Let's Encrypt cert expired`
- `/nginx:diagnose rate limit is blocking real users`
