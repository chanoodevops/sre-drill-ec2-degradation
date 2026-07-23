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

### 13.6 Enterprise Hardening (Drop-in Layer)

The base Option 2 unit is sufficient for staging. Production deployments add a **drop-in override** — the base unit stays untouched, and hardening directives layer on top. This pattern survives package upgrades and config management runs.

#### 13.6.1 Create Drop-in Directory

```bash
sudo mkdir -p /etc/systemd/system/storage-breaker.service.d
sudo tee /etc/systemd/system/storage-breaker.service.d/enterprise-hardening.conf <<'EOF'
[Service]
# --- Resource Limits ---
MemoryMax=512M
MemoryHigh=384M
CPUQuota=50%
TasksMax=32
LimitNOFILE=4096

# --- Seccomp ---
# Block ~300+ syscalls (mount, reboot, kexec, module load, etc.)
SystemCallFilter=@system-service
# Block 32-bit syscall vectors (defense-in-depth)
SystemCallArchitectures=native

# --- Crash-loop Protection ---
# Stop restarting after 3 failures in 60 seconds
StartLimitIntervalSec=60
StartLimitBurst=3
# Notify monitoring on permanent failure
StartLimitAction=none

# --- Read-only Code Mount ---
# App code is read-only at runtime — compromise cannot modify files
BindReadOnlyPaths=/opt/storage-breaker
# Writable tmpfs for runtime data (PID files, Unix sockets, etc.)
RuntimeDirectory=storage-breaker
RuntimeDirectoryMode=0750
EOF
```

**Directive reference:**

| Directive | Value | Effect |
|-----------|-------|--------|
| `MemoryMax` | `512M` | Hard OOM kill at 512 MB RSS |
| `MemoryHigh` | `384M` | Soft throttle above 384 MB |
| `CPUQuota` | `50%` | Maximum 1/2 CPU core |
| `TasksMax` | `32` | Prevents fork bomb — cap processes per unit |
| `LimitNOFILE` | `4096` | File descriptor ceiling for connection bursts |
| `SystemCallFilter` | `@system-service` | Blocks ~300+ syscalls (mount, reboot, kexec, bpf) |
| `SystemCallArchitectures` | `native` | Blocks 32-bit syscall vectors (defense-in-depth) |
| `StartLimitIntervalSec` | `60` | Window for restart counting |
| `StartLimitBurst` | `3` | Max restarts in window before permanent failure |
| `BindReadOnlyPaths` | `/opt/storage-breaker` | Remounts app dir read-only at runtime |
| `RuntimeDirectory` | `storage-breaker` | Creates `/run/storage-breaker` with lifecycle tied to unit |

#### 13.6.2 Apply Drop-in

```bash
sudo systemctl daemon-reload
sudo systemctl restart storage-breaker
```

Verify overrides applied:

```bash
systemctl show storage-breaker | grep -E '(MemoryMax|CPUQuota|TasksMax)' | sort
```

Expected:

```
CPUQuotaPerSecUSec=50000
MemoryMax=536870912
TasksMax=32
```

Verify read-only mount:

```bash
mount | grep storage-breaker
```

Expected:

```
/dev/nvme0n1p1 on /opt/storage-breaker type ext4 (ro,...)
```

#### 13.6.3 Log Retention (logrotate)

systemd journal is a transport, not a retention policy. Enterprise configures logrotate:

```bash
sudo tee /etc/logrotate.d/storage-breaker <<'EOF'
/var/log/storage-breaker/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 0640 storage-breaker adm
    sharedscripts
    postrotate
        systemctl kill -s USR1 storage-breaker 2>/dev/null || true
    endscript
}
EOF
```

Test configuration:

```bash
sudo logrotate -d /etc/logrotate.d/storage-breaker
```

Expected output — dry-run, shows rotation path without executing:

```
rotating pattern: /var/log/storage-breaker/*.log  after 1 days (30 rotations)
empty log files are not rotated, old logs are removed
considering log /var/log/storage-breaker/access.log
  log does not need rotating (log has been rotated at 2026-07-22 12:00)
```

| Directive | Reason |
|-----------|--------|
| `rotate 30` | 30-day retention — standard compliance minimum |
| `compress` | gzip rotated logs — reduces archival size ~90% |
| `create 0640 storage-breaker adm` | Perms survive rotation — no ownership drift |
| `sharedscripts` | One `postrotate` call per run, not per file |

> **Storage cost:** 100 MB/day logs compress to ~10 MB. 30-day retention = ~300 MB journal + ~300 MB compressed archive. Budget 1 GB for log storage on the root volume.

#### 13.6.4 Audit Trail (auditd)

Forensic trail for the service account — mandatory for SOC-2, ISO 27001, PCI-DSS:

```bash
sudo tee /etc/audit/rules.d/storage-breaker.rules <<'EOF'
-w /opt/storage-breaker -p wa -k storage-breaker-code
-w /var/log/storage-breaker -p wa -k storage-breaker-logs
EOF

sudo auditctl -R /etc/audit/rules.d/storage-breaker.rules
```

Verify rules loaded:

```bash
sudo auditctl -l | grep storage-breaker
```

Expected:

```
-w /opt/storage-breaker -p wa -k storage-breaker-code
-w /var/log/storage-breaker -p wa -k storage-breaker-logs
```

Search audit log for service activity:

```bash
sudo ausearch -k storage-breaker-code --just-one
```

#### 13.6.5 Enterprise Comparison

| Control | Base Option 2 | Enterprise Option 2 |
|---------|---------------|---------------------|
| Code mutation protection | Writable (`ReadWritePaths`) | Read-only (`BindReadOnlyPaths`) |
| Memory limit | None | `MemoryMax=512M`, `MemoryHigh=384M` |
| CPU limit | None | `CPUQuota=50%` |
| Task limit | None | `TasksMax=32` |
| FD limit | systemd default (usually 4096) | Explicit `LimitNOFILE=4096` |
| Seccomp | None | `SystemCallFilter=@system-service` + native arch |
| Crash-loop cap | Infinite restart | 3 failures in 60s → permanent failed state |
| Log retention | journal only (no rotation) | logrotate: daily, 30-day, gzip |
| Forensic audit | None | auditd rules per path |
| Config method | Edits unit file directly | Drop-in override (survives updates) |

> **Cost note:** auditd incurs ~1–3% CPU overhead on write-heavy paths. For a single-service EC2 instance, negligible. For large fleets, consider forwarding audit events to CloudWatch Logs with a 7-day retention filter to avoid unbounded journal growth.

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

### Option 2 — Enterprise Hardening

Same as Option 2 plus:

- [ ] Drop-in directory created at `/etc/systemd/system/storage-breaker.service.d/`
- [ ] Resource limits applied (`MemoryMax=512M`, `CPUQuota=50%`, `TasksMax=32`)
- [ ] Seccomp filter enabled (`SystemCallFilter=@system-service`)
- [ ] Crash-loop protection configured (`StartLimitBurst=3` in 60s)
- [ ] App directory remounted read-only at runtime (`BindReadOnlyPaths`)
- [ ] logrotate rule installed for `/var/log/storage-breaker/*.log`
- [ ] auditd rules loaded for service paths
- [ ] Drop-in verified with `systemctl show`

---

## See Also

| Document | Description |
|----------|-------------|
| [API Reference](api-reference.md) | Application endpoint documentation |
| [Systemd Guide](systemd-guide.md) | Service management, logging, and security |
| [Nginx Guide](nginx-guide.md) | Reverse proxy, HTTPS, Let's Encrypt, ACM |
| [Health Check Runbook](health-check-runbook.md) | Troubleshooting, RCA, and recovery procedures |
| [Disk Exhaustion Report](disk-exhaustion-report.md) | Enterprise guide: detection, recovery, and prevention |
