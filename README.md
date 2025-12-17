# Production HTTPS Node.js API on AWS EC2 (Nginx + Let’s Encrypt)

This project demonstrates a **production-grade Node.js API deployment** on AWS EC2 using **Nginx as a reverse proxy**, **HTTPS with Let’s Encrypt**, and a **secure network design**.

The focus is not just “making it work”, but **debugging real-world issues** and implementing industry-standard practices.

---

## Architecture Overview

Internet
│
│ HTTP (80) / HTTPS (443)
▼
Nginx (Reverse Proxy + TLS Termination)
│
│ Internal traffic only
▼
Node.js (Express)
127.0.0.1:3000

---

## Key Design Decisions (WHY)

- **Application port (3000) is NOT public**
  - Only accessible via `127.0.0.1`
  - Prevents direct external access to the app
- **Nginx handles TLS termination**
  - Industry standard for scalability and security
- **HTTP → HTTPS enforced**
  - Prevents insecure access
- **Let’s Encrypt with ACME HTTP-01**
  - Free, trusted certificates
  - Automated renewal
- **Subdomain isolation**
  - `api.vokario.com` isolated from `www.vokario.com`

---

## Tech Stack

- AWS EC2 (Ubuntu 24.04)
- Nginx (Reverse Proxy)
- Node.js (Express)
- Let’s Encrypt (Certbot)
- systemd (Node.js service)
- AWS Security Groups

---

## Networking & Security

### AWS Security Group
| Port | Purpose |
|----|----|
| 80 | HTTP → HTTPS redirect + ACME challenge |
| 443 | HTTPS (TLS) |
| 3000 | ❌ Closed (internal only) |

### Security Highlights
- No public access to application port
- TLS 1.2 / 1.3 enabled
- HSTS enabled
- Basic rate limiting on Nginx
- SSH not required (SSM supported)

---

## Nginx Configuration (Production)

```nginx
limit_req_zone $binary_remote_addr zone=api_ratelimit:10m rate=10r/s;

server {
    listen 80;
    server_name api.vokario.com;

    location ^~ /.well-known/acme-challenge/ {
        root /var/www/letsencrypt;
        default_type "text/plain";
        allow all;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name api.vokario.com;

    ssl_certificate     /etc/letsencrypt/live/api.vokario.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.vokario.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;

    access_log /var/log/nginx/api.access.log;
    error_log  /var/log/nginx/api.error.log;

    location / {
        limit_req zone=api_ratelimit burst=20 nodelay;

        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
Let’s Encrypt (ACME HTTP-01)
Certificate obtained using webroot method:


sudo certbot certonly \
  --webroot \
  -w /var/www/letsencrypt \
  -d api.vokario.com
Auto-Renewal
Managed by systemd timer

Zero-downtime renewal


sudo certbot renew --dry-run
Node.js (Reverse Proxy Awareness)
To correctly detect HTTPS behind Nginx:


app.set('trust proxy', true);
This prevents incorrect HTTP assumptions and browser security warnings.

Verification
curl -I http://api.vokario.com
# 301 → HTTPS

curl -I https://api.vokario.com
# 200 OK
Browser:

🔒 Secure

Valid Let’s Encrypt certificate

Real Production Debugging Covered
ACME challenge 404 behind reverse proxy

Nginx sites-available vs sites-enabled issue

Port 443 blocked by Security Group

Browser HSTS / cache causing false “Not Secure” warning

HTTPS awareness behind proxy (trust proxy)

CV-Ready Summary
Deployed Node.js API behind Nginx reverse proxy on AWS EC2

Implemented HTTPS using Let’s Encrypt (ACME HTTP-01 challenge)

Enforced HTTP → HTTPS redirection with TLS termination

Hardened Nginx with security headers and rate limiting

Debugged real-world SSL, ACME, and browser HSTS issues

Next Steps
CloudWatch Logs & Metrics (observability)

ALB + Target Groups

Auto Scaling & high availability
