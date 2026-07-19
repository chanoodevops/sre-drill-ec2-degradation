# Systemd Service Management Guide

Complete guide for managing the Storage Breaker application as a systemd service.

---

## What is systemd?

systemd is the init system and service manager for Linux. It starts services at boot, manages their lifecycle, and provides logging via journald.

**Why use systemd for this application:**
- Auto-restart on failure
- Start at boot automatically
- Centralized logging
- Process isolation and security

---

## Service File Location

```
/etc/systemd/system/storage-breaker.service
```

---

## Service File Anatomy

```ini
[Unit]
Description=Storage Breaker FastAPI Service    # Human-readable description
After=network.target                           # Start after network is ready

[Service]
Type=simple                                    # Process type

User=ubuntu                                    # Run as this user
Group=ubuntu                                   # Run as this group

WorkingDirectory=/opt/storage-breaker          # App directory

ExecStart=/opt/storage-breaker/.venv/bin/uvicorn app:app \
    --host 127.0.0.1 \
    --port 3000 \
    --workers 1 \
    --no-access-log

Restart=always                                 # Auto-restart on exit
RestartSec=5                                   # Wait 5s before restart

Environment=PYTHONUNBUFFERED=1                 # Environment variables

StandardOutput=journal                         # Log stdout to journald
StandardError=journal                          # Log stderr to journald

NoNewPrivileges=true                           # Security: no privilege escalation
PrivateTmp=true                                # Security: private /tmp

[Install]
WantedBy=multi-user.target                     # Start at multi-user boot
```

### Key Directives

| Directive | Purpose |
|-----------|---------|
| `Type=simple` | ExecStart is the main process |
| `Restart=always` | Restart on any exit (crash, clean exit, etc.) |
| `RestartSec=5` | Delay 5 seconds between restarts |
| `NoNewPrivileges=true` | Prevent privilege escalation attacks |
| `PrivateTmp=true` | Isolate /tmp for this service |

---

## Common Commands

### Service Lifecycle

| Command | Description |
|---------|-------------|
| `sudo systemctl start storage-breaker` | Start the service |
| `sudo systemctl stop storage-breaker` | Stop the service |
| `sudo systemctl restart storage-breaker` | Restart the service |
| `sudo systemctl reload storage-breaker` | Reload config (if supported) |
| `sudo systemctl status storage-breaker` | Check current status |

### Enable at Boot

| Command | Description |
|---------|-------------|
| `sudo systemctl enable storage-breaker` | Start automatically at boot |
| `sudo systemctl disable storage-breaker` | Don't start at boot |

### Reload systemd

After modifying the service file:

```bash
sudo systemctl daemon-reload
```

---

## Viewing Logs

### Using journalctl

| Command | Description |
|---------|-------------|
| `sudo journalctl -u storage-breaker` | All logs for the service |
| `sudo journalctl -u storage-breaker -f` | Follow logs in real-time |
| `sudo journalctl -u storage-breaker -n 50` | Last 50 lines |
| `sudo journalctl -u storage-breaker --since "1 hour ago"` | Logs from last hour |
| `sudo journalctl -u storage-breaker --since "2026-07-17"` | Logs from specific date |
| `sudo journalctl -u storage-breaker -p err` | Only error messages |
| `sudo journalctl -u storage-breaker --no-pager` | Don't paginate output |

### Filter by Priority

| Priority | Meaning |
|----------|---------|
| `emerg` | System is unusable |
| `alert` | Action must be taken immediately |
| `crit` | Critical conditions |
| `err` | Error conditions |
| `warning` | Warning conditions |
| `notice` | Normal but significant |
| `info` | Informational |
| `debug` | Debug-level messages |

Example:

```bash
sudo journalctl -u storage-breaker -p warning..err
```

---

## Troubleshooting

### Service Won't Start

```bash
# Check status and recent logs
sudo systemctl status storage-breaker

# Check for syntax errors in service file
sudo systemd-analyze verify /etc/systemd/system/storage-breaker.service

# Check detailed error
sudo journalctl -u storage-breaker -n 20 --no-pager
```

**Common causes:**
- Incorrect `ExecStart` path
- Missing dependencies
- Permission denied
- Port already in use

### Service Keeps Restarting

```bash
# Check restart count
sudo systemctl show storage-breaker -p NRestarts

# Check logs for crash reason
sudo journalctl -u storage-breaker -n 100 --no-pager | grep -i "error\|exception\|traceback"
```

**Common causes:**
- Application error on startup
- Missing Python dependencies
- Configuration file missing

### Service Stops but Doesn't Restart

```bash
# Check if Restart=always is set
sudo systemctl show storage-breaker | grep Restart

# Check exit status
sudo journalctl -u storage-breaker -n 10 --no-pager
```

### Permission Denied Errors

```bash
# Check service user/group
sudo systemctl show storage-breaker -p User -p Group

# Check file permissions
ls -la /opt/storage-breaker
ls -la /var/log/storage-breaker

# Fix ownership
sudo chown -R ubuntu:ubuntu /opt/storage-breaker
sudo chown -R ubuntu:ubuntu /var/log/storage-breaker
```

### Port Already in Use

```bash
# Check what's using port 3000
sudo ss -lntp | grep :3000

# Kill the conflicting process
sudo kill $(sudo lsof -t -i :3000)
```

---

## Resource Limits

### Add Resource Limits to Service File

```ini
[Service]
# ... existing directives ...

# Memory limit
MemoryMax=512M
MemoryHigh=256M

# CPU limit (100% = 1 CPU core)
CPUQuota=50%

# File descriptor limit
LimitNOFILE=65536

# Process limit
LimitNPROC=4096
```

### Apply Changes

```bash
sudo systemctl daemon-reload
sudo systemctl restart storage-breaker
```

---

## Security Hardening

### Recommended Security Directives

```ini
[Service]
# ... existing directives ...

# Security options
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/log/storage-breaker
ReadWritePaths=/opt/storage-breaker
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
RestrictNamespaces=true
RestrictRealtime=true
RestrictSUIDSGID=true
MemoryDenyWriteExecute=true
```

### What These Do

| Directive | Protection |
|-----------|------------|
| `ProtectSystem=strict` | Make / and /usr read-only |
| `ProtectHome=true` | Make /home inaccessible |
| `ReadWritePaths=...` | Allow writes only to specific paths |
| `ProtectKernelTunables=true` | Prevent modifying /proc, /sys |
| `RestrictNamespaces=true` | Prevent creating namespaces |
| `MemoryDenyWriteExecute=true` | Prevent W+X memory mappings |

---

## Auto-Restart Behavior

### Restart Modes

| Value | Behavior |
|-------|----------|
| `no` | Don't restart (default) |
| `on-success` | Restart only on clean exit (code 0) |
| `on-failure` | Restart on non-zero exit or signal |
| `on-abnormal` | Restart on timeout, watchdog, or signal |
| `on-abort` | Restart on unclean signal |
| `always` | Always restart |

### Restart Limits

```ini
[Service]
Restart=always
RestartSec=5
StartLimitIntervalSec=300
StartLimitBurst=5
```

This means: if the service fails 5 times within 5 minutes, stop restarting.

---

## Environment Variables

### Set in Service File

```ini
[Service]
Environment=PYTHONUNBUFFERED=1
Environment=APP_ENV=production
Environment=DATABASE_URL=postgresql://localhost/db
```

### Load from File

```ini
[Service]
EnvironmentFile=/etc/storage-breaker/env
```

Example `/etc/storage-breaker/env`:

```
PYTHONUNBUFFERED=1
APP_ENV=production
SECRET_KEY=your-secret-here
```

---

## systemd Timer (Alternative to Cron)

If you need periodic tasks instead of a long-running service:

### Create Timer Unit

```bash
sudo nano /etc/systemd/system/storage-cleanup.timer
```

```ini
[Unit]
Description=Run storage cleanup daily

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

### Create Service Unit

```bash
sudo nano /etc/systemd/system/storage-cleanup.service
```

```ini
[Unit]
Description=Storage Breaker cleanup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/cleanup-logs.sh
User=ubuntu
```

### Enable and Start

```bash
sudo systemctl daemon-reload
sudo systemctl enable storage-cleanup.timer
sudo systemctl start storage-cleanup.timer

# Check timer status
sudo systemctl list-timers --all
```

---

## Quick Reference Card

```
START        sudo systemctl start storage-breaker
STOP         sudo systemctl stop storage-breaker
RESTART      sudo systemctl restart storage-breaker
STATUS       sudo systemctl status storage-breaker
ENABLE       sudo systemctl enable storage-breaker
DISABLE      sudo systemctl disable storage-breaker
LOGS         sudo journalctl -u storage-breaker -f
RELOAD       sudo systemctl daemon-reload
```

---

## See Also

| Document | Description |
|----------|-------------|
| [Deployment Guide](deployment.md) | Infrastructure setup and application deployment |
| [API Reference](api-reference.md) | Application endpoint documentation |
| [Health Check Runbook](health-check-runbook.md) | Troubleshooting, RCA, and recovery procedures |
| [Disk Exhaustion Report](disk-exhaustion-report.md) | Enterprise guide: detection, recovery, and prevention |
