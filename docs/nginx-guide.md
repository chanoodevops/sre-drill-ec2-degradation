# Nginx Reverse Proxy Configuration Guide

Complete guide for configuring Nginx as a reverse proxy for the Storage Breaker application, including HTTP, HTTPS with Let's Encrypt, and AWS Certificate Manager (ACM) integration.

---

## Overview

```
Client (HTTP/HTTPS)
        │
        ▼
    Nginx (Port 80/443)
        │
        ▼
Uvicorn (127.0.0.1:3000)
```

**Why Nginx?**
- Terminates SSL/TLS connections
- Handles HTTP/2 and HTTP/3
- Load balancing (if scaling)
- Static file serving
- Rate limiting and security headers

---

## Prerequisites

```bash
# Install Nginx
sudo apt update
sudo apt install -y nginx

# Verify installation
nginx -v

# Start and enable
sudo systemctl start nginx
sudo systemctl enable nginx
```

---

## Option 1: HTTP Only (Development/Testing)

### Basic Configuration

```bash
sudo nano /etc/nginx/sites-available/storage-breaker
```

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name _;

    # Logs
    access_log /var/log/nginx/storage-breaker-access.log;
    error_log /var/log/nginx/storage-breaker-error.log;

    # Reverse proxy
    location / {
        proxy_pass http://127.0.0.1:3000;

        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 10s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # Health check endpoint (optional: serve directly from Nginx)
    location /health {
        proxy_pass http://127.0.0.1:3000/health;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Enable and Test

```bash
# Create symlink
sudo ln -s /etc/nginx/sites-available/storage-breaker \
    /etc/nginx/sites-enabled/

# Remove default site
sudo rm -f /etc/nginx/sites-enabled/default

# Test configuration
sudo nginx -t

# Reload
sudo systemctl reload nginx
```

### Verify

```bash
curl -i http://localhost/health
curl -i http://YOUR_EC2_IP/health
```

---

## Option 2: HTTPS with Let's Encrypt (Recommended for Production)

### Prerequisites

- Domain name pointing to your EC2 public IP
- Port 80 and 443 open in Security Group

### Step 1: Install Certbot

```bash
sudo apt install -y certbot python3-certbot-nginx
```

### Step 2: Initial HTTP Configuration

Create the Nginx config with your domain:

```bash
sudo nano /etc/nginx/sites-available/storage-breaker
```

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name your-domain.com www.your-domain.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/storage-breaker /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### Step 3: Obtain SSL Certificate

```bash
sudo certbot --nginx -d your-domain.com -d www.your-domain.com
```

Follow the prompts:
1. Enter email address
2. Agree to terms of service
3. Choose whether to redirect HTTP to HTTPS

### Step 4: Verify Auto-Renewal

```bash
# Test renewal process
sudo certbot renew --dry-run

# Check certificate expiry
sudo certbot certificates
```

### Step 5: Final HTTPS Configuration

After Certbot, your config will be automatically updated:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name your-domain.com www.your-domain.com;

    # Redirect all HTTP to HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name your-domain.com www.your-domain.com;

    # SSL Certificate (managed by Certbot)
    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;

    # Include Certbot's recommended SSL settings
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    # Logs
    access_log /var/log/nginx/storage-breaker-access.log;
    error_log /var/log/nginx/storage-breaker-error.log;

    # Security Headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Reverse proxy
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # Cache static assets (if any)
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        proxy_pass http://127.0.0.1:3000;
        expires 7d;
        add_header Cache-Control "public, immutable";
    }
}
```

### Step 6: Auto-Renewal with systemd

Certbot installs a systemd timer for auto-renewal:

```bash
# Check timer status
sudo systemctl status certbot.timer

# View renewal logs
sudo journalctl -u certbot
```

Or create a cron job as backup:

```bash
sudo crontab -e
```

Add:

```bash
0 0,12 * * * certbot renew --quiet --post-hook "systemctl reload nginx"
```

---

## Option 3: HTTPS with AWS Certificate Manager (ACM)

ACM is used with AWS load balancers (ALB/NLB), not directly with Nginx on EC2. Use this if you have an ALB in front of your EC2.

### Architecture

```
Client (HTTPS)
        │
        ▼
    ALB (ACM Certificate)
        │
        ▼
    EC2 (Nginx - HTTP only)
        │
        ▼
Uvicorn (127.0.0.1:3000)
```

### Step 1: Request Certificate in ACM

```bash
# Using AWS CLI
aws acm request-certificate \
    --domain-name your-domain.com \
    --subject-alternative-names www.your-domain.com \
    --validation-method DNS
```

Or via AWS Console:
1. Go to **Certificate Manager**
2. Click **Request a certificate**
3. Enter domain name
4. Choose **DNS validation**
5. Add CNAME records to your DNS

### Step 2: Create ALB

```bash
# Create target group
aws elbv2 create-target-group \
    --name storage-breaker-tg \
    --protocol HTTP \
    --port 80 \
    --vpc-id vpc-XXXXXXXX \
    --target-type instance \
    --health-check-path /health \
    --health-check-interval-seconds 30 \
    --health-check-timeout-seconds 5 \
    --healthy-threshold-count 2 \
    --unhealthy-threshold-count 3

# Register EC2 instance
aws elbv2 register-targets \
    --target-group-arn arn:aws:elasticloadbalancing:REGION:ACCOUNT:targetgroup/storage-breaker-tg/ID \
    --targets Id=i-XXXXXXXXXXXXXXXXX

# Create ALB
aws elbv2 create-load-balancer \
    --name storage-breaker-alb \
    --subnets subnet-XXXXXXXX subnet-YYYYYYYY \
    --security-groups sg-XXXXXXXX \
    --type application \
    --scheme internet-facing

# Create HTTPS listener with ACM certificate
aws elbv2 create-listener \
    --load-balancer-arn arn:aws:elasticloadbalancing:REGION:ACCOUNT:loadbalancer/app/storage-breaker-alb/ID \
    --protocol HTTPS \
    --port 443 \
    --certificates CertificateArn=arn:aws:acm:REGION:ACCOUNT:certificate/CERTIFICATE_ID \
    --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:REGION:ACCOUNT:targetgroup/storage-breaker-tg/ID

# Create HTTP to HTTPS redirect
aws elbv2 create-listener \
    --load-balancer-arn arn:aws:elasticloadbalancing:REGION:ACCOUNT:loadbalancer/app/storage-breaker-alb/ID \
    --protocol HTTP \
    --port 80 \
    --default-actions Type=redirect,RedirectConfig='{Protocol=HTTPS,Port=443,StatusCode=HTTP_301}'
```

### Step 3: Nginx Configuration (Behind ALB)

When behind ALB, Nginx only needs HTTP:

```nginx
server {
    listen 80;

    server_name your-domain.com;

    # ALB handles SSL termination
    # Trust X-Forwarded-* headers from ALB
    set_real_ip_from 10.0.0.0/8;
    real_ip_header X-Forwarded-For;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## Option 4: Hybrid (Let's Encrypt on EC2 + ALB for Scaling)

Use Let's Encrypt directly on EC2 for single-instance deployments, or ACM with ALB for multi-instance setups.

| Scenario | Solution |
|----------|----------|
| Single EC2, direct access | Let's Encrypt on Nginx |
| Multiple EC2, load balanced | ACM on ALB |
| Staging + Production | Let's Encrypt (staging) + ACM (prod) |

---

## Security Headers

Add these to your Nginx config for production:

```nginx
# Prevent clickjacking
add_header X-Frame-Options DENY always;

# Prevent MIME type sniffing
add_header X-Content-Type-Options nosniff always;

# XSS protection
add_header X-XSS-Protection "1; mode=block" always;

# HSTS (only add after confirming HTTPS works)
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

# Content Security Policy (adjust for your needs)
add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'" always;

# Referrer Policy
add_header Referrer-Policy "strict-origin-when-cross-origin" always;

# Permissions Policy
add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
```

---

## Rate Limiting

Protect against abuse:

```nginx
# Define rate limit zone (in http block or server block)
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

server {
    # ... other config ...

    # Apply rate limiting to health endpoint
    location /health {
        limit_req zone=api burst=20 nodelay;
        proxy_pass http://127.0.0.1:3000/health;
    }

    # Apply rate limiting to all other routes
    location / {
        limit_req zone=api burst=5 nodelay;
        proxy_pass http://127.0.0.1:3000;
        # ... proxy headers ...
    }
}
```

---

## Gzip Compression

Enable compression for better performance:

```nginx
server {
    # ... other config ...

    # Enable gzip
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml
        application/rss+xml
        image/svg+xml;
}
```

---

## WebSocket Support (If Needed)

If your application uses WebSockets:

```nginx
location /ws {
    proxy_pass http://127.0.0.1:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_read_timeout 86400;
}
```

---

## Troubleshooting

### 502 Bad Gateway

```bash
# Check if Uvicorn is running
pgrep -af uvicorn

# Check if port 3000 is listening
sudo ss -lntp | grep :3000

# Check Nginx error log
sudo tail -20 /var/log/nginx/storage-breaker-error.log
```

### 504 Gateway Timeout

```bash
# Increase timeouts in Nginx config
proxy_connect_timeout 30s;
proxy_send_timeout 120s;
proxy_read_timeout 120s;

# Check if app is slow
curl -w "@curl-format.txt" -o /dev/null -s http://127.0.0.1:3000/health
```

### SSL Certificate Issues

```bash
# Check certificate expiry
sudo certbot certificates

# Force renewal
sudo certbot renew --force-renewal

# Check Nginx SSL config
sudo nginx -t

# View SSL errors
sudo journalctl -u nginx | grep -i ssl
```

### Permission Denied

```bash
# Check Nginx can read certificate
sudo ls -la /etc/letsencrypt/live/your-domain.com/

# Fix permissions
sudo chmod 600 /etc/letsencrypt/live/your-domain.com/privkey.pem
sudo chown root:root /etc/letsencrypt/live/your-domain.com/*
```

---

## Quick Reference

```
TEST CONFIG      sudo nginx -t
RELOAD           sudo systemctl reload nginx
RESTART          sudo systemctl restart nginx
VIEW LOGS        sudo tail -f /var/log/nginx/storage-breaker-error.log
CHECK STATUS     sudo systemctl status nginx
RENEW CERTS      sudo certbot renew
CHECK CERTS      sudo certbot certificates
```

---

## See Also

| Document | Description |
|----------|-------------|
| [Deployment Guide](deployment.md) | Infrastructure setup and application deployment |
| [Systemd Guide](systemd-guide.md) | Service management, logging, and security |
| [API Reference](api-reference.md) | Application endpoint documentation |
| [Health Check Runbook](health-check-runbook.md) | Troubleshooting, RCA, and recovery procedures |
| [Disk Exhaustion Report](disk-exhaustion-report.md) | Enterprise guide: detection, recovery, and prevention |
