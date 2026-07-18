# Troubleshooting Guide

Procedures for diagnosing and resolving issues with the Storage Breaker application.

## Common Issues

### Health Check Returns 503

**Symptoms:**

```bash
curl -i http://EC2_PUBLIC_IP/health
# HTTP/1.1 503 Service Unavailable
# {"status":"unhealthy"}
```

**Diagnosis:**

1. Check disk usage:

```bash
df -h /var/log/storage-breaker
```

If usage is ≥ 95%, the health endpoint returns unhealthy.

2. Check application logs:

```bash
sudo tail -20 /var/log/storage-breaker/application.log
```

3. Check service status:

```bash
sudo systemctl status storage-breaker
```

**Resolution:** See [Recovery Procedures](#recovery-procedures) below.

---

### SSH Connection Fails

**Symptoms:**

```bash
ssh ubuntu@EC2_PUBLIC_IP
# Connection timed out
```

**Diagnosis:**

1. Verify instance is running in AWS Console
2. Check Security Group allows port 22 from your IP
3. Verify correct PEM key:

```bash
ssh -i <correct-key>.pem ubuntu@EC2_PUBLIC_IP
```

---

### Nginx 502 Bad Gateway

**Symptoms:**

```bash
curl -i http://EC2_PUBLIC_IP/health
# HTTP/1.1 502 Bad Gateway
```

**Diagnosis:**

1. Check if Uvicorn is running:

```bash
pgrep -af uvicorn
```

2. Check Nginx error log:

```bash
sudo tail -20 /var/log/nginx/storage-breaker-error.log
```

3. Check port binding:

```bash
sudo ss -lntp | grep -E ':80|:3000'
```

**Resolution:**

```bash
sudo systemctl restart storage-breaker
sudo systemctl restart nginx
```

---

### Service Fails to Start

**Symptoms:**

```bash
sudo systemctl status storage-breaker
# Active: failed
```

**Diagnosis:**

```bash
sudo journalctl -xeu storage-breaker
```

**Common causes:**

- Missing virtual environment
- Incorrect file permissions
- Port 3000 already in use

**Resolution:**

```bash
# Check permissions
sudo chown -R ubuntu:ubuntu /opt/storage-breaker

# Check port
sudo ss -lntp | grep :3000

# Restart
sudo systemctl restart storage-breaker
```

---

## Recovery Procedures

### Emergency Recovery (Disk Full)

Use when `/dev/root` is at or near 100% usage.

**Step 1: Stop the application**

```bash
sudo systemctl stop storage-breaker
```

Confirm stopped:

```bash
pgrep -af uvicorn
```

**Step 2: Check log file size**

```bash
sudo ls -lh /var/log/storage-breaker/application.log
sudo du -sh /var/log/storage-breaker
```

**Step 3: Truncate the log file**

Truncating preserves the file and permissions while removing contents:

```bash
sudo truncate -s 0 /var/log/storage-breaker/application.log
```

Alternatively:

```bash
sudo sh -c '> /var/log/storage-breaker/application.log'
```

**Step 4: Verify recovered space**

```bash
df -h /
```

Example:

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/root        30G   10G   19G  35% /
```

**Step 5: Restart the application**

```bash
sudo systemctl start storage-breaker
sudo systemctl status storage-breaker
```

**Step 6: Verify health**

```bash
curl -i http://EC2_PUBLIC_IP/health
```

Expected:

```
HTTP/1.1 200 OK
{"status":"healthy"}
```

---

### Delete Log File

Use when the application can safely recreate the file.

```bash
sudo systemctl stop storage-breaker

sudo rm -f /var/log/storage-breaker/application.log

sudo touch /var/log/storage-breaker/application.log
sudo chown ubuntu:ubuntu /var/log/storage-breaker/application.log
sudo chmod 640 /var/log/storage-breaker/application.log

sudo systemctl start storage-breaker
```

### Truncate vs Delete

| Action | Recommendation |
| ------ | -------------- |
| `truncate -s 0` | Preferred when application has file open |
| `rm` | Use when application is stopped |
| Delete while running | Avoid — space may not release until process closes file |

Check for deleted-but-open files:

```bash
sudo lsof +L1
```

---

## Advanced Recovery

### Use Dedicated EBS Volume

Recommended architecture for production:

```
Root EBS volume (/dev/root)
├── Operating system
├── Nginx
└── Python application

Dedicated log EBS volume (/dev/nvme1n1)
└── /var/log/storage-breaker
```

See [Deployment Guide](deployment.md) for EBS volume setup instructions.

### Increase Root EBS Volume

When more storage is required on the root filesystem:

1. Modify volume in AWS Console
2. Extend partition:

```bash
sudo growpart /dev/nvme0n1 1
```

3. Extend filesystem:

```bash
# For ext4
sudo resize2fs /dev/nvme0n1p1

# For XFS
sudo xfs_growfs -d /
```

4. Verify:

```bash
df -h /
```

---

### Configure Log Rotation

For normal applications where logs should not grow indefinitely:

```bash
sudo nano /etc/logrotate.d/storage-breaker
```

Configuration:

```
/var/log/storage-breaker/application.log {
    size 100M
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    create 0640 ubuntu ubuntu
}
```

| Setting | Purpose |
| ------- | ------- |
| `size 100M` | Rotate at 100 MB |
| `rotate 7` | Keep 7 old files |
| `compress` | Compress rotated logs |
| `copytruncate` | Copy then truncate active file |

Test:

```bash
sudo logrotate -d /etc/logrotate.d/storage-breaker
```

Force rotation:

```bash
sudo logrotate -f /etc/logrotate.d/storage-breaker
```

---

### Add Disk Usage Safety Threshold

Prevent complete disk exhaustion by stopping writes before 100%:

```python
import shutil
from pathlib import Path

LOG_DIRECTORY = Path("/var/log/storage-breaker")
STOP_WRITING_THRESHOLD_PERCENT = 92.0


def get_disk_usage_percent() -> float:
    usage = shutil.disk_usage(LOG_DIRECTORY)
    if usage.total == 0:
        return 0.0
    return (usage.used / usage.total) * 100


def should_stop_writing() -> bool:
    return get_disk_usage_percent() >= STOP_WRITING_THRESHOLD_PERCENT
```

---

## Recovery Checklist

- [ ] Stop the Storage Breaker service
- [ ] Confirm no log-generation process remains
- [ ] Check root filesystem usage
- [ ] Check application log size
- [ ] Truncate or remove the application log
- [ ] Check for deleted but open files (`lsof +L1`)
- [ ] Confirm disk space recovered
- [ ] Start the application
- [ ] Test health endpoint
- [ ] Monitor log growth
- [ ] Confirm exactly one Uvicorn worker

---

## Monitoring Commands Reference

| Command | Purpose |
| ------- | ------- |
| `df -h /` | Check root filesystem |
| `df -h /var/log/storage-breaker` | Check log filesystem |
| `ls -lh /var/log/storage-breaker/application.log` | Check log file size |
| `pgrep -af uvicorn` | Verify worker count |
| `sudo ss -lntp \| grep -E ':80\|:3000'` | Check port bindings |
| `sudo systemctl status storage-breaker` | Check service status |
| `curl -s http://127.0.0.1:3000/health` | Test health endpoint |
| `sudo lsof +L1` | Find deleted open files |
| `df -ih` | Check inode usage |
