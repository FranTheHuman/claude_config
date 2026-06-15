# /nginx:harden — Apply Security Hardening to Nginx

Run a complete security hardening audit and generate the fixes.
Covers CVE patching, TLS hardening, security headers, rate limiting,
Fail2ban setup, and firewall recommendations.
Apply all rules from `.claude/skills/nginx-ops/SKILL.md`.

## Instructions

1. Read `.claude/skills/nginx-ops/SKILL.md` sections 2, 3, 6, 7, 8.
2. Ask for the Nginx version (`nginx -v`) if not provided.
3. Check CVE-2026-42945 exposure first.
4. Run the full checklist below.
5. Output issues by severity with exact fixes.

## Hardening Checklist

### 1. CVE-2026-42945 (NGINX Rift)
```bash
nginx -v  # affected: 0.6.27–1.30.0
grep -rn "rewrite.*\$[0-9].*?" /etc/nginx/
```
- If version is affected AND rewrite pattern found → `[CRITICAL]` — patch immediately
- If version is affected AND no rewrite pattern → `[WARNING]` — update anyway

### 2. Version hiding
```bash
grep "server_tokens" /etc/nginx/nginx.conf
```
- Missing or not `off` → `[CRITICAL]`

### 3. TLS protocols
```bash
grep "ssl_protocols" /etc/nginx/conf.d/*.conf /etc/nginx/nginx.conf
openssl s_client -connect localhost:443 -tls1    # must fail
openssl s_client -connect localhost:443 -tls1_1  # must fail
```
- TLS 1.0 or 1.1 present → `[CRITICAL]`
- Missing `ssl_protocols` directive → `[WARNING]`

### 4. Cipher suites
```bash
grep "ssl_ciphers" /etc/nginx/conf.d/*.conf
grep "ssl_prefer_server_ciphers" /etc/nginx/conf.d/*.conf
```
- Weak ciphers (MD5, RC4, 3DES) → `[CRITICAL]`
- Missing `ssl_prefer_server_ciphers on` → `[WARNING]`

### 5. Security headers
```bash
curl -k -I https://localhost | grep -i "strict\|x-frame\|x-content\|x-xss"
```
- Missing HSTS → `[CRITICAL]`
- Missing X-Frame-Options → `[WARNING]`
- Missing X-Content-Type-Options → `[WARNING]`

### 6. Rate limiting
```bash
grep "limit_req" /etc/nginx/conf.d/*.conf
grep "limit_req_zone" /etc/nginx/nginx.conf
```
- No rate limit on login/auth endpoints → `[CRITICAL]`
- No global rate limiting → `[WARNING]`

### 7. Fail2ban
```bash
systemctl is-active fail2ban
fail2ban-client status nginx-http-auth 2>/dev/null
```
- Not installed or not monitoring Nginx → `[WARNING]`

### 8. Firewall — backend ports
```bash
ss -tlnp | grep -E "3000|8080|8082"
```
- Backend ports publicly accessible → `[CRITICAL]`

## Output Structure

### Security Score: X/10

### Issues

```
[SEVERITY] Title
Check:  Command that revealed the issue
Problem: What risk this creates
Fix:    Exact config or command to resolve
```

### Hardening Commands (run in order)

Complete sequence to apply all fixes:

```bash
# All fix commands in the correct order
sudo nginx -t && sudo systemctl reload nginx
```

## Arguments

Usage: `/nginx:harden` — no arguments, audits current Nginx setup.

Options:
- `/nginx:harden --fix-only` → skip analysis, generate only the fix configs
- `/nginx:harden --cve-only` → check only for CVE-2026-42945 exposure
- `/nginx:harden --tls-only` → audit only TLS configuration
