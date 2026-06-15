# /nginx:setup — Generate Complete Nginx Proxy Setup for the shalapp Stack

Generate the full, hardened Nginx configuration for the current shalapp deployment:
API (Node.js), web frontend, backoffice frontend, and MinIO public assets.
Includes TLS configuration, security headers, rate limiting, and Let's Encrypt setup.
Apply all rules from `.claude/skills/nginx-ops/SKILL.md`.

## Instructions

1. Read `.claude/skills/nginx-ops/SKILL.md` sections 2–5 before generating.
2. Ask for the domain name if not provided (e.g., `yourdomain.com`).
3. Generate ALL files listed below — complete, not stubs.
4. Always generate snippets first (they are shared by all virtual hosts).
5. Include the installation and verification commands at the end.

## Generated Files

### Shared snippets (generate first)

- `/etc/nginx/snippets/ssl-params.conf` — TLS 1.2/1.3, cipher suites, OCSP, DH params
- `/etc/nginx/snippets/security-headers.conf` — HSTS (300s to start), X-Frame, nosniff
- `/etc/nginx/snippets/proxy-params.conf` — headers, timeouts, hide backend info

### nginx.conf

- Global hardening: `server_tokens off`, buffer limits, timeouts
- Rate limit zone definitions: api, login, uploads, conn_limit
- Gzip compression
- SSL session cache (shared across all vhosts)

### Virtual host configs

- `00-redirect.conf` — HTTP → HTTPS + Let's Encrypt ACME challenge path
- `01-api.conf` — api.DOMAIN → shalapp-api:3000 with auth + upload rate limits
- `02-web.conf` — DOMAIN → shalapp-web:8080
- `03-backoffice.conf` — backoffice.DOMAIN → shalapp-backoffice:8082
- `04-storage.conf` — storage.DOMAIN → MinIO:9000 (public paths only, 403 on rest)

### Setup commands

```bash
# 1. Generate DH params (run once, takes a few minutes)
sudo openssl dhparam -out /etc/nginx/dhparam.pem 2048

# 2. Install certbot
sudo apt install certbot python3-certbot-nginx -y

# 3. First: deploy config WITHOUT SSL certs (comment out ssl lines)
#    Run certbot to get certs, then uncomment SSL
sudo certbot --nginx \
    -d DOMAIN \
    -d www.DOMAIN \
    -d api.DOMAIN \
    -d backoffice.DOMAIN \
    -d storage.DOMAIN

# 4. Test and reload
sudo nginx -t && sudo systemctl reload nginx

# 5. Verify TLS
openssl s_client -connect DOMAIN:443 -tls1   # must FAIL
openssl s_client -connect DOMAIN:443 -tls1_3 # must succeed

# 6. Verify headers
curl -I https://DOMAIN | grep -i "strict\|x-frame\|x-content\|server:"

# 7. After 24h of stable HTTPS — increase HSTS to 1 year
# Edit /etc/nginx/snippets/security-headers.conf:
# max-age=31536000
```

## Arguments

Usage: `/nginx:setup [domain]`

Examples:
- `/nginx:setup` → asks for domain, generates full setup
- `/nginx:setup yourdomain.com` → generates everything for that domain
- `/nginx:setup yourdomain.com --with-ip-restrict-backoffice` → adds IP restriction to backoffice
