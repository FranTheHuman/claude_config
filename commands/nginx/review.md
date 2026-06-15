# /nginx:review — Review Nginx Configuration for Security and Correctness

Review an Nginx configuration file or the full setup for security issues,
missing hardening, wrong routing, and deprecated patterns.
Apply all rules from `.claude/skills/nginx-ops/SKILL.md`.

## Instructions

1. Read `.claude/skills/nginx-ops/SKILL.md` sections 2, 3, 7 before reviewing.
2. Read the target config file(s). If no path given, ask which file to review.
3. Check immediately for hard rule violations — instant CRITICAL:
   - `server_tokens` not set to `off`
   - TLS 1.0 or 1.1 present in `ssl_protocols`
   - Missing security headers (HSTS, X-Frame-Options, X-Content-Type-Options)
   - `proxy_buffering` not disabled for MinIO locations
   - Backend ports exposed as upstreams without firewall mention
4. Check CVE-2026-42945: look for rewrite rules with unnamed captures + `?`.
5. Check rate limiting on auth endpoints.
6. Output the review in the structure below.

## Output Structure

### Summary
One sentence on overall security posture and the most critical gap.

### Issues

Group by severity:

```
[SEVERITY] Short title
Location: file:section
Problem:  What is wrong and why it matters for security or reliability.
Fix:      Exact corrected Nginx directive or config block.
```

Severity levels:
- `[CRITICAL]` — exploitable, audit failure, or will break the service
- `[WARNING]`  — weak configuration, missing hardening, performance risk
- `[SUGGESTION]` — improvement to clarity, monitoring, or future-proofing

### Verdict
- ✅ Approved — hardened and correct
- ⚠️ Approved with warnings — functional but needs improvement
- ❌ Blocked — critical issues must be fixed before production

## Arguments

Usage: `/nginx:review [path/to/config]`

Examples:
- `/nginx:review /etc/nginx/nginx.conf`
- `/nginx:review /etc/nginx/conf.d/01-api.conf`
- `/nginx:review /etc/nginx/conf.d/` → review all virtual host configs
- `/nginx:review` → review whatever config is in context
