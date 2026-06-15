---
name: nginx-ops
description: >
  Expert operational knowledge for running Nginx as a production reverse proxy on a
  small VPS using Docker Compose. Covers hardened configuration with TLS 1.2/1.3,
  security headers (HSTS, X-Frame-Options, CSP, X-Content-Type-Options), rate limiting,
  reverse proxy setup for Node.js APIs, MinIO S3 storage, and static frontends,
  Let's Encrypt certificate automation, CVE monitoring and patching, performance tuning
  for constrained environments, Fail2ban integration, log analysis, and multi-service
  routing patterns. Designed to proxy the shalapp stack: shalapp-api (Node.js),
  shalapp-web (frontend), shalapp-backoffice (frontend), and MinIO (object storage).
---

You are an expert Nginx operator for small VPS deployments. Apply this skill as the
authoritative source for every Nginx decision. The guiding principle:
**minimal attack surface, maximum reliability — Nginx is the only thing the internet touches**.

---

## 1. Current Deployment Context

```
VPS Stack (Docker Compose — shalapp):
├── Nginx (THIS AGENT'S DOMAIN)          ← single entry point from internet
│   ├── → shalapp-api    :3000           ← Node.js API
│   ├── → shalapp-web    :8080           ← React frontend
│   ├── → shalapp-backoffice :8082       ← React backoffice
│   └── → minio          :9000           ← MinIO S3 API (public paths only)
├── shalapp-api      (Node.js, port 3000)
├── shalapp-web      (frontend, port 8080)
├── shalapp-backoffice (frontend, port 8082)
├── minio            (S3, port 9000 loopback only)
├── shalapp-db       (PostgreSQL, internal only)
└── shalapp-redis    (Redis, internal only)

Current exposure:
- Ports 80/443 → Nginx (public)
- Port 9000/9002 → MinIO (127.0.0.1 only, SSH tunnel for console)
- Port 3000 → API (currently exposed, should be internal only via Nginx)
- Port 5433 → PostgreSQL (127.0.0.1 only)
- Port 6380 → Redis (127.0.0.1 only)
```

---

## 2. Hardened Base Configuration

### `/etc/nginx/nginx.conf` — global hardening

```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
    worker_connections 1024;
    multi_accept on;
}

http {
    # ─── Security: hide version ───────────────────────────────────────
    server_tokens off;

    # ─── MIME types ───────────────────────────────────────────────────
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # ─── Performance ──────────────────────────────────────────────────
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    # ─── Buffer limits (DDoS mitigation) ──────────────────────────────
    client_body_buffer_size    16k;
    client_header_buffer_size   1k;
    client_max_body_size       50m;   # raise for file upload endpoints
    large_client_header_buffers 4 8k;

    # ─── Timeouts ─────────────────────────────────────────────────────
    client_body_timeout   12;
    client_header_timeout 12;
    send_timeout          10;

    # ─── Rate limiting zones ──────────────────────────────────────────
    limit_req_zone $binary_remote_addr zone=api:10m    rate=30r/s;
    limit_req_zone $binary_remote_addr zone=login:10m  rate=5r/m;
    limit_req_zone $binary_remote_addr zone=uploads:10m rate=10r/m;
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

    # ─── Logging ──────────────────────────────────────────────────────
    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    'rt=$request_time';

    access_log /var/log/nginx/access.log main;
    error_log  /var/log/nginx/error.log warn;

    # ─── TLS session cache (shared across all virtual hosts) ──────────
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 1h;
    ssl_session_tickets off;

    # ─── Gzip compression ─────────────────────────────────────────────
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript
               text/xml application/xml application/xml+rss text/javascript;

    # ─── Virtual host configs ─────────────────────────────────────────
    include /etc/nginx/conf.d/*.conf;
}
```

---

## 3. TLS Configuration (2026 Standard)

### SSL parameters snippet — `/etc/nginx/snippets/ssl-params.conf`

```nginx
# Modern TLS only — no TLS 1.0 or 1.1
ssl_protocols TLSv1.2 TLSv1.3;

# Strong cipher suites — server decides, not client
ssl_ciphers 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH';
ssl_prefer_server_ciphers on;

# OCSP stapling — faster cert validation
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout 5s;

# Diffie-Hellman for forward secrecy (generate once)
# sudo openssl dhparam -out /etc/nginx/dhparam.pem 2048
ssl_dhparam /etc/nginx/dhparam.pem;
```

### Security headers snippet — `/etc/nginx/snippets/security-headers.conf`

```nginx
# HSTS — force HTTPS for 1 year (start with max-age=300 and increase gradually)
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

# Clickjacking protection
add_header X-Frame-Options "SAMEORIGIN" always;

# MIME-sniffing protection
add_header X-Content-Type-Options "nosniff" always;

# XSS protection (legacy browsers)
add_header X-XSS-Protection "1; mode=block" always;

# Referrer policy
add_header Referrer-Policy "strict-origin-when-cross-origin" always;

# Remove backend identity headers
proxy_hide_header X-Powered-By;
proxy_hide_header Server;
```

---

## 4. Service Routing — Complete Configuration

### HTTP → HTTPS redirect — `/etc/nginx/conf.d/00-redirect.conf`

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name _;

    # Allow Let's Encrypt ACME challenge
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    # Redirect everything else to HTTPS
    location / {
        return 301 https://$host$request_uri;
    }
}
```

### Main API — `/etc/nginx/conf.d/01-api.conf`

```nginx
upstream shalapp_api {
    server 127.0.0.1:3000;         # or shalapp-api:3000 if Nginx is in Docker
    keepalive 32;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name api.yourdomain.com;

    ssl_certificate     /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    include snippets/ssl-params.conf;
    include snippets/security-headers.conf;

    # Rate limiting
    limit_req zone=api burst=50 nodelay;
    limit_conn conn_limit 20;

    # Auth endpoints — tighter rate limit
    location ~ ^/api/(auth|login|register) {
        limit_req zone=login burst=3 nodelay;
        proxy_pass http://shalapp_api;
        include snippets/proxy-params.conf;
    }

    # Upload endpoints — larger body, upload rate limit
    location ~ ^/api/(upload|files) {
        limit_req zone=uploads burst=5 nodelay;
        client_max_body_size 100m;
        proxy_request_buffering off;    # required for streaming uploads
        proxy_pass http://shalapp_api;
        include snippets/proxy-params.conf;
    }

    # General API
    location /api/ {
        proxy_pass http://shalapp_api;
        include snippets/proxy-params.conf;
    }

    # Block access to anything not under /api/
    location / {
        return 404;
    }
}
```

### Frontend — `/etc/nginx/conf.d/02-web.conf`

```nginx
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name yourdomain.com www.yourdomain.com;

    ssl_certificate     /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    include snippets/ssl-params.conf;
    include snippets/security-headers.conf;

    location / {
        proxy_pass http://127.0.0.1:8080;
        include snippets/proxy-params.conf;
    }
}
```

### Backoffice — `/etc/nginx/conf.d/03-backoffice.conf`

```nginx
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name backoffice.yourdomain.com;

    ssl_certificate     /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    include snippets/ssl-params.conf;
    include snippets/security-headers.conf;

    # Optional: restrict access to known IPs
    # allow 203.0.113.0/24;  # your office IP
    # deny all;

    location / {
        proxy_pass http://127.0.0.1:8082;
        include snippets/proxy-params.conf;
    }
}
```

### MinIO public assets — `/etc/nginx/conf.d/04-storage.conf`

```nginx
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name storage.yourdomain.com;

    ssl_certificate     /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    include snippets/ssl-params.conf;
    include snippets/security-headers.conf;

    # MinIO requires these for streaming uploads/downloads
    client_max_body_size 0;
    proxy_buffering off;
    proxy_request_buffering off;

    # Public paths — logos, blog images (anonymous access)
    location ~ ^/shalapp/(logo|blog)/ {
        proxy_pass http://127.0.0.1:9000;
        include snippets/proxy-params.conf;

        # Cache public assets aggressively
        proxy_cache_valid 200 7d;
        add_header Cache-Control "public, max-age=604800, immutable";
    }

    # Block everything else — private objects need pre-signed URLs
    location / {
        return 403;
    }
}
```

### Proxy parameters snippet — `/etc/nginx/snippets/proxy-params.conf`

```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_cache_bypass $http_upgrade;

proxy_read_timeout  90s;
proxy_connect_timeout 10s;
proxy_send_timeout  90s;

proxy_hide_header X-Powered-By;
proxy_hide_header Server;
```

---

## 5. Let's Encrypt with Certbot

### Initial certificate

```bash
# Install certbot
sudo apt install certbot python3-certbot-nginx -y

# Obtain certificates for all domains
sudo certbot --nginx \
    -d yourdomain.com \
    -d www.yourdomain.com \
    -d api.yourdomain.com \
    -d backoffice.yourdomain.com \
    -d storage.yourdomain.com

# Verify auto-renewal works
sudo certbot renew --dry-run
```

### Auto-renewal cron (if not already set by certbot)

```bash
# Check if certbot timer exists
systemctl status certbot.timer

# If not, add to cron:
echo "0 0 * * * root certbot renew --quiet --post-hook 'systemctl reload nginx'" \
    | sudo tee /etc/cron.d/certbot-renew
```

---

## 6. Rate Limiting Reference

### Zone definitions (in nginx.conf http block)

```nginx
# General API — 30 requests/second per IP
limit_req_zone $binary_remote_addr zone=api:10m rate=30r/s;

# Auth/login — 5 requests/minute per IP (brute force protection)
limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;

# Uploads — 10 requests/minute per IP
limit_req_zone $binary_remote_addr zone=uploads:10m rate=10r/m;

# Connection limit
limit_conn_zone $binary_remote_addr zone=conn_limit:10m;
```

### Apply in location blocks

```nginx
# Burst allows temporary spikes, nodelay serves burst immediately
limit_req zone=api burst=50 nodelay;
limit_req zone=login burst=3 nodelay;
limit_conn conn_limit 20;

# Return 429 instead of 503 for rate limited requests
limit_req_status 429;
limit_conn_status 429;
```

---

## 7. Security Hardening Checklist

### CVE-2026-42945 (NGINX Rift) — check if affected

```bash
# Check your Nginx version
nginx -v

# Affected: 0.6.27 – 1.30.0
# Fixed:    1.30.1+ or 1.31.0+

# Check for vulnerable rewrite patterns in config
grep -rn "rewrite.*\$[0-9].*?" /etc/nginx/
# If found: replace unnamed captures ($1, $2) with named captures (?P<name>...)

# Update immediately if on affected version
sudo apt update && sudo apt upgrade nginx
sudo systemctl restart nginx
```

### Configuration audit commands

```bash
# Test configuration syntax
sudo nginx -t

# Check active TLS protocols (should fail for TLS 1.0/1.1)
openssl s_client -connect yourdomain.com:443 -tls1   # should FAIL
openssl s_client -connect yourdomain.com:443 -tls1_1 # should FAIL
openssl s_client -connect yourdomain.com:443 -tls1_2 # should succeed
openssl s_client -connect yourdomain.com:443 -tls1_3 # should succeed

# Check security headers are present
curl -k -I https://yourdomain.com | grep -i "strict\|x-frame\|x-content\|server:"

# Check version is hidden
curl -I https://yourdomain.com | grep "Server:"
# Should show "Server: nginx" (no version number)
```

---

## 8. Fail2ban Integration

### Install and configure for Nginx

```bash
sudo apt install fail2ban -y
```

### `/etc/fail2ban/jail.local`

```ini
[DEFAULT]
bantime  = 3600
findtime = 600
maxretry = 5
action = %(action_mwl)s

[nginx-http-auth]
enabled  = true
port     = http,https
logpath  = /var/log/nginx/error.log

[nginx-botsearch]
enabled  = true
port     = http,https
logpath  = /var/log/nginx/access.log
maxretry = 2

[nginx-req-limit]
enabled  = true
port     = http,https
logpath  = /var/log/nginx/error.log
maxretry = 10
findtime = 60
```

### Fail2ban commands

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status
sudo fail2ban-client status nginx-http-auth
sudo fail2ban-client set nginx-http-auth unbanip 1.2.3.4  # unban an IP
```

---

## 9. Operations

### Daily commands

```bash
# Check Nginx status
sudo systemctl status nginx

# Test config before reload (always do this first)
sudo nginx -t && sudo systemctl reload nginx

# View live access log
sudo tail -f /var/log/nginx/access.log

# View errors only
sudo tail -f /var/log/nginx/error.log | grep -v "^$"

# Check which IPs are hitting hardest
sudo awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -20

# Check which endpoints are most requested
sudo awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -20

# Check for 4xx/5xx errors
sudo awk '$9 >= 400 {print $9, $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -20
```

### Update Nginx safely

```bash
# Check current version
nginx -v

# Update
sudo apt update && sudo apt upgrade nginx

# Test config, then restart
sudo nginx -t && sudo systemctl restart nginx

# Verify new version
nginx -v
```

---

## 10. Troubleshooting

### Common errors

| Error | Cause | Fix |
|-------|-------|-----|
| `502 Bad Gateway` | Backend not running or wrong port | Check backend container, verify upstream address |
| `413 Request Entity Too Large` | `client_max_body_size` too small | Increase for upload endpoints: `client_max_body_size 100m` |
| `504 Gateway Timeout` | Backend too slow | Increase `proxy_read_timeout` for slow endpoints |
| `403 Forbidden` | Path blocked by `deny all` or location config | Check location blocks for deny rules |
| `SSL handshake failed` | TLS version mismatch or bad cert | Check `ssl_protocols`, verify cert with `openssl s_client` |
| `upstream timed out` | Backend overloaded or unreachable | Check backend health, Docker network, increase timeout |

### Diagnose a 502

```bash
# 1. Is the backend container running?
docker ps | grep shalapp-api

# 2. Is it responding on its port?
curl -s http://127.0.0.1:3000/health

# 3. Check Nginx error log
sudo tail -20 /var/log/nginx/error.log

# 4. Check if Nginx can reach backend
sudo -u www-data curl -s http://127.0.0.1:3000/health

# 5. If using Docker network, verify connectivity
docker exec nginx-container curl -s http://shalapp-api:3000/health
```

---

## 11. Adding a New Service (Future Growth)

When adding a new backend service to proxy through Nginx:

```bash
# 1. Create config file
sudo nano /etc/nginx/conf.d/05-newservice.conf

# 2. Standard template for a new service
cat > /etc/nginx/conf.d/05-newservice.conf << 'EOF'
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name newservice.yourdomain.com;

    ssl_certificate     /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    include snippets/ssl-params.conf;
    include snippets/security-headers.conf;

    limit_req zone=api burst=30 nodelay;

    location / {
        proxy_pass http://127.0.0.1:NEW_PORT;
        include snippets/proxy-params.conf;
    }
}
EOF

# 3. Add domain to Let's Encrypt cert
sudo certbot certonly --nginx -d newservice.yourdomain.com

# 4. Test and reload
sudo nginx -t && sudo systemctl reload nginx
```

---

## 12. Architecture Decision Guide

| Question | Answer |
|----------|--------|
| Should ports 3000/8080/8082 be public? | No — close them in firewall, route only through Nginx |
| HTTP/2 always? | Yes — add `http2` to all `listen 443` directives |
| Separate SSL cert per domain? | No — wildcard cert covers all subdomains |
| Start HSTS max-age? | 300 (5 min) → verify → increase to 31536000 |
| MinIO console through Nginx? | No — SSH tunnel is sufficient and safer |
| Cache static frontend assets? | Yes — use `proxy_cache` or let the frontend set Cache-Control |
| Rate limit all endpoints equally? | No — auth endpoints need tighter limits than API |
| Nginx in Docker or on host? | On host (or in Docker with `network_mode: host`) — simpler routing |
| When to add WAF (ModSecurity)? | When you need OWASP rule-based filtering — adds complexity |

---

## Reference

- Mozilla SSL Config Generator: https://ssl-config.mozilla.org/
- NGINX docs: https://nginx.org/en/docs/
- Let's Encrypt: https://letsencrypt.org/
- Fail2ban: https://www.fail2ban.org/
- CVE-2026-42945 advisory: check https://nginx.org/en/security_advisories.html
