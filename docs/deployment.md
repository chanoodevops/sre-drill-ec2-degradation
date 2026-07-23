# Deployment Guide

Complete instructions for deploying the Storage Breaker application on an AWS EC2 instance.

## Prerequisites

- AWS EC2 Ubuntu instance (22.04 LTS or 24.04 LTS)
- Security Group allowing:
  - TCP 22 (SSH) from your IP
  - TCP 80 (HTTP) from your IP or client network
- PEM private key file
- Git repository access

## 1. Connect to EC2

On your local machine:

```bash
chmod 400 <your-key>.pem

eval "$(ssh-agent -s)"
ssh-add <your-key>.pem

ssh -i <your-key>.pem ubuntu@EC2_PUBLIC_IP
```

Verify connection:

```bash
whoami
hostname
```

## 2. Update System

```bash
sudo apt update
sudo apt upgrade -y
```

Install required packages:

```bash
sudo apt install -y \
    python3 \
    python3-venv \
    python3-pip \
    git \
    nginx \
    curl
```

Verify installations:

```bash
python3 --version
git --version
nginx -v
```

## 3. Clone Repository

Create application directory:

```bash
sudo mkdir -p /opt/storage-breaker
sudo chown -R ubuntu:ubuntu /opt/storage-breaker
```

Clone:

```bash
git clone <repository-url> /opt/storage-breaker
cd /opt/storage-breaker
```

Verify:

```bash
ls -la
```

Expected output:

```
app.py
requirements.txt
```

## 4. Create Log Directory

```bash
sudo mkdir -p /var/log/storage-breaker
sudo chown -R ubuntu:ubuntu /var/log/storage-breaker
sudo chmod 750 /var/log/storage-breaker
```

Verify:

```bash
ls -ld /var/log/storage-breaker
```

Expected: `drwxr-x---` permissions with `ubuntu` owner and group.

Verify:

```bash
ls -ld /var/log/storage-breaker
```

## 5. Create Python Virtual Environment

```bash
cd /opt/storage-breaker

python3 -m venv .venv
source .venv/bin/activate

python -m pip install --upgrade pip
python -m pip install -r requirements.txt

deactivate
```

Verify:

```bash
/opt/storage-breaker/.venv/bin/python --version
```

## 6. Test Application Manually

Start the application:

```bash
cd /opt/storage-breaker

.venv/bin/uvicorn app:app \
    --host 127.0.0.1 \
    --port 3000 \
    --workers 1 \
    --no-access-log
```

In a separate terminal, test:

```bash
curl -i http://127.0.0.1:3000/health
```

Expected response:

```
HTTP/1.1 200 OK
{"status":"healthy"}
```

Stop with `Ctrl + C` when testing is complete.

## 7. Create systemd Service

Create the service file:

```bash
sudo nano /etc/systemd/system/storage-breaker.service
```

Paste the following configuration:

```ini
[Unit]
Description=Storage Breaker FastAPI Service
After=network.target

[Service]
Type=simple

User=ubuntu
Group=ubuntu

WorkingDirectory=/opt/storage-breaker

ExecStart=/opt/storage-breaker/.venv/bin/uvicorn app:app \
    --host 127.0.0.1 \
    --port 3000 \
    --workers 1 \
    --no-access-log

Restart=always
RestartSec=5

Environment=PYTHONUNBUFFERED=1

StandardOutput=journal
StandardError=journal

NoNewPrivileges=true
PrivateTmp=true

# Log directory permissions are set during deployment (Step 4)
# Application logs are created with 640 permissions

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable storage-breaker
sudo systemctl start storage-breaker
```

Verify:

```bash
sudo systemctl status storage-breaker
```

View logs:

```bash
sudo journalctl -u storage-breaker -f
```

## 8. Configure Nginx

Create site configuration:

```bash
sudo nano /etc/nginx/sites-available/storage-breaker
```

Paste:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name _;

    access_log /var/log/nginx/storage-breaker-access.log;
    error_log /var/log/nginx/storage-breaker-error.log;

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
}
```

Enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/storage-breaker \
    /etc/nginx/sites-enabled/storage-breaker

sudo rm -f /etc/nginx/sites-enabled/default
```

Validate and restart:

```bash
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl enable nginx
```

## 9. Configure Security Group

In AWS Console:

| Type | Port | Source |
| ---- | ---- | ------ |
| SSH | 22 | Your IP |
| HTTP | 80 | Client IP |

> **Important:** Do not expose port 3000 publicly.

## 10. Configure UFW (Optional)

If UFW is enabled:

```bash
sudo ufw allow OpenSSH
sudo ufw allow "Nginx Full"
sudo ufw reload
```

## 11. Verify Deployment

Test through Uvicorn:

```bash
curl -i http://127.0.0.1:3000/health
```

Test through Nginx:

```bash
curl -i http://127.0.0.1/health
```

Test externally:

```bash
curl -i http://EC2_PUBLIC_IP/health
```

## 12. Verify Worker Count

```bash
pgrep -af uvicorn
```

Or:

```bash
ps -ef | grep uvicorn
```

Confirm exactly **one** worker process is running.

---

## 13. Alternative: Dedicated Service Account (Production Standard)

The default deployment runs the app as the `ubuntu` SSH user. This section documents the **production standard**: a dedicated system account with no login, no home directory, and no sudo.

### 13.1 Create Service Account

Run this **before cloning the repository** (before Step 3):

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin storage-breaker
```

| Flag | Meaning |
|------|---------|
| `--system` | UID < 1000, no password expiry |
| `--no-create-home` | No `/home/storage-breaker` |
| `--shell /usr/sbin/nologin` | Cannot log in, cannot `su` directly |

Verify:

```bash
id storage-breaker
```

Expected:

```
uid=998(storage-breaker) gid=998(storage-breaker) groups=998(storage-breaker)
```

### 13.2 Fix App Directory Ownership

After cloning (Step 3), change ownership so `ubuntu` owns files but only the service account can run them:

```bash
sudo chown -R ubuntu:storage-breaker /opt/storage-breaker
sudo chmod 750 /opt/storage-breaker
sudo chmod 750 /opt/storage-breaker/.venv/bin/python
```

| Owner | Group | Perms | Result |
|-------|-------|-------|--------|
| `ubuntu` | `storage-breaker` | `750` | Deployer writes code via git. Service reads + executes. World blocked. |

Verify:

```bash
ls -ld /opt/storage-breaker
```

Expected:

```
drwxr-x--- 3 ubuntu storage-breaker 4096 Jul 23 12:00 /opt/storage-breaker
```

### 13.3 Fix Log Directory Ownership

After creating the log directory (Step 4), change to service account:

```bash
sudo chown storage-breaker:adm /var/log/storage-breaker
sudo chmod 750 /var/log/storage-breaker
```

Group `adm` lets operators read logs without `sudo`.

Verify:

```bash
ls -ld /var/log/storage-breaker
```

Expected:

```
drwxr-x--- 2 storage-breaker adm 4096 Jul 23 12:00 /var/log/storage-breaker
```

### 13.4 systemd Service File

Replace the service file in Step 7 with this hardened version:

```ini
[Unit]
Description=Storage Breaker FastAPI Service
After=network.target

[Service]
Type=simple

User=storage-breaker
Group=storage-breaker
SupplementaryGroups=

WorkingDirectory=/opt/storage-breaker

ExecStart=/opt/storage-breaker/.venv/bin/uvicorn app:app \
    --host 127.0.0.1 \
    --port 3000 \
    --workers 1 \
    --no-access-log

Restart=always
RestartSec=5

Environment=PYTHONUNBUFFERED=1

StandardOutput=journal
StandardError=journal

# --- Hardening ---
NoNewPrivileges=true
PrivateTmp=true

# Drop all Linux capabilities
CapabilityBoundingSet=

# Read-only system paths
ProtectSystem=strict
ProtectHome=true

# Only these paths are writable
ReadWritePaths=/var/log/storage-breaker /opt/storage-breaker

[Install]
WantedBy=multi-user.target
```

**Key hardening directives:**

| Directive | Effect |
|-----------|--------|
| `SupplementaryGroups=` | Empty — app inherits no extra groups |
| `CapabilityBoundingSet=` | Cannot bind low ports, change ownership, or any privileged syscall |
| `ProtectSystem=strict` | `/usr`, `/etc` read-only; whitelisted paths via `ReadWritePaths` |
| `ProtectHome=true` | Cannot read `/home/ubuntu` — SSH keys stay safe |
| `ReadWritePaths=` | Explicit whitelist — everything else is read-only |

After updating, reload and restart:

```bash
sudo systemctl daemon-reload
sudo systemctl restart storage-breaker
```

Verify process runs as `storage-breaker`:

```bash
ps -ef | grep uvicorn | grep -v grep
```

Expected:

```
storage-+ 12345     1  0 12:00 ?        00:00:01 /opt/storage-breaker/.venv/bin/python ...
```

### 13.5 Security Comparison

| Control | Option 1 (`ubuntu` user) | Option 2 (`storage-breaker`) |
|---------|--------------------------|------------------------------|
| SSH key exposure risk | App compromise reads `/home/ubuntu/.ssh` | `ProtectHome=true` blocks access |
| Code tampering | App can modify its own source | App cannot write to `750` dirs |
| Privilege escalation | `NoNewPrivileges=true` only | Same + empty capabilities |
| Audit attribution | All actions as `ubuntu` | Distinct UID per service |
| Log readability | `ubuntu` or `sudo` only | Group `adm` — no sudo needed |
| Day-2 ops friction | Low — one user for SSH + deploy | Moderate — `sudo` for file changes |

> **When to use which:** Option 1 for labs, dev, and training drills. Option 2 for production deployments. This drill scenario uses Option 1 for clarity, but teams should default to Option 2 in live environments.

---

## Deployment Checklist

### Option 1 (Development / Training)

- [ ] Ubuntu updated
- [ ] Python, Git, Nginx installed
- [ ] Repository cloned to `/opt/storage-breaker`
- [ ] Virtual environment created
- [ ] Dependencies installed
- [ ] Log directory created at `/var/log/storage-breaker`
- [ ] Application tested locally
- [ ] systemd service created and enabled
- [ ] Nginx configured and validated
- [ ] Security Group allows ports 22 and 80
- [ ] Health endpoint returns `200 OK`
- [ ] Exactly one Uvicorn worker running

### Option 2 (Production)

Same as Option 1 plus:

- [ ] Service account `storage-breaker` created before clone
- [ ] App directory ownership set to `ubuntu:storage-breaker` (750)
- [ ] Log directory ownership set to `storage-breaker:adm` (750)
- [ ] systemd unit updated with hardened directives (`CapabilityBoundingSet=`, `ProtectSystem=strict`, `ProtectHome=true`)
- [ ] Process confirmed running as `storage-breaker` user

---

## See Also

| Document | Description |
|----------|-------------|
| [API Reference](api-reference.md) | Application endpoint documentation |
| [Systemd Guide](systemd-guide.md) | Service management, logging, and security |
| [Nginx Guide](nginx-guide.md) | Reverse proxy, HTTPS, Let's Encrypt, ACM |
| [Health Check Runbook](health-check-runbook.md) | Troubleshooting, RCA, and recovery procedures |
| [Disk Exhaustion Report](disk-exhaustion-report.md) | Enterprise guide: detection, recovery, and prevention |
