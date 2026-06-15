# /nginx:add-service — Add a New Service to the Nginx Proxy

Generate the complete Nginx config to proxy a new backend service.
Includes TLS, security headers, rate limiting, and Let's Encrypt domain addition.
Apply all rules from `.claude/skills/nginx-ops/SKILL.md`.

## Instructions

1. Read `.claude/skills/nginx-ops/SKILL.md` section 11 (Adding a New Service).
2. Parse the arguments: service name, subdomain, backend port, service type.
3. Select the rate limiting profile based on service type (API, frontend, storage).
4. Generate the complete config file with the next available number prefix.
5. Include the certbot command to add the domain to the existing cert.
6. Include test and reload commands.

## Service type profiles

| Type | Rate limit | Special config |
|------|-----------|---------------|
| `api` | `zone=api burst=50` + auth endpoints get `zone=login` | `proxy_read_timeout 90s` |
| `frontend` | `zone=api burst=30` | Standard proxy |
| `storage` | No rate limit on GET | `proxy_buffering off`, `client_max_body_size 0` |
| `admin` | `zone=api burst=10` | Optional IP restriction |
| `websocket` | `zone=api burst=100` | `Upgrade` + `Connection` headers |

## Output Structure

### New config file: `/etc/nginx/conf.d/NN-servicename.conf`

```nginx
# Complete, copy-paste ready config
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name subdomain.yourdomain.com;

    ssl_certificate     /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    include snippets/ssl-params.conf;
    include snippets/security-headers.conf;

    # Rate limiting appropriate for this service type
    limit_req zone=api burst=30 nodelay;

    location / {
        proxy_pass http://127.0.0.1:PORT;
        include snippets/proxy-params.conf;
    }
}
```

### Commands to deploy

```bash
# 1. Create the config file
sudo nano /etc/nginx/conf.d/NN-servicename.conf
# (paste config above)

# 2. Add domain to existing Let's Encrypt cert
sudo certbot certonly --nginx -d subdomain.yourdomain.com

# 3. Test syntax
sudo nginx -t

# 4. Reload (if test passes)
sudo systemctl reload nginx

# 5. Verify
curl -I https://subdomain.yourdomain.com
```

## Arguments

Usage: `/nginx:add-service <name> <subdomain> <port> [--type api|frontend|storage|admin|websocket]`

Examples:
- `/nginx:add-service notifications notif.yourdomain.com 4000 --type websocket`
- `/nginx:add-service reports reports.yourdomain.com 5000 --type api`
- `/nginx:add-service docs docs.yourdomain.com 3010 --type frontend`
- `/nginx:add-service admin-panel panel.yourdomain.com 8090 --type admin`
