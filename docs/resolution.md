# Resolution Guide: EC2 Disk Exhaustion Failure

## Failure Summary

| Field | Detail |
| ----- | ------ |
| **Incident** | Health check returns `503 Service Unavailable` |
| **Root Cause** | Disk exhaustion on `/dev/root` — application log file consumed all available space |
| **Impact** | Application unhealthy, Nginx returning 503, risk of OS instability |
| **Severity** | High — service degraded, potential cascade to SSH/Nginx/systemd failures |
| **Affected Component** | `/var/log/storage-breaker/application.log` on root EBS volume |

---

## Temporary Resolution (Emergency Recovery)

Use this when the root filesystem is already at or near 100%. The goal is to restore service as quickly as possible.

### Step 1: Stop the Application

Prevent further disk consumption before attempting recovery.

```bash
sudo systemctl stop storage-breaker
```

Verify the process has stopped:

```bash
pgrep -af uvicorn
# Expected: no output (no uvicorn processes running)
```

### Step 2: Assess Disk Usage

Document the state before making changes:

```bash
df -h /
sudo ls -lh /var/log/storage-breaker/application.log
sudo du -sh /var/log/storage-breaker
```

Example state:

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/root       8.7G  8.6G   83M 100% /

-rw-r----- 1 ubuntu ubuntu 8.2G Jul 17 14:12 application.log
```

### Step 3: Free Disk Space

**Option A — Truncate (preferred when file is open or permissions must be preserved):**

```bash
sudo truncate -s 0 /var/log/storage-breaker/application.log
```

**Option B — Delete and recreate (use when application is stopped):**

```bash
sudo rm -f /var/log/storage-breaker/application.log
sudo touch /var/log/storage-breaker/application.log
sudo chown ubuntu:ubuntu /var/log/storage-breaker/application.log
sudo chmod 640 /var/log/storage-breaker/application.log
```

### Step 4: Verify Space Recovered

```bash
df -h /
```

Expected:

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/root       8.7G  2.4G  6.3G  28% /
```

### Step 5: Check for Deleted-but-Open Files

If a file was deleted while a process still held it open, disk space may not be released:

```bash
sudo lsof +L1
```

If processes appear holding deleted files, restart them:

```bash
sudo systemctl restart storage-breaker
```

### Step 6: Restart the Application

```bash
sudo systemctl start storage-breaker
sudo systemctl status storage-breaker
```

### Step 7: Verify Health Restored

```bash
curl -i http://EC2_PUBLIC_IP/health
```

Expected:

```
HTTP/1.1 200 OK
{"status":"healthy"}
```

### Step 8: Confirm Worker Count

Exactly one Uvicorn worker must be running:

```bash
pgrep -af uvicorn
# Expected: one line showing uvicorn with --workers 1
```

---

## Long-Term Resolution (Permanent Fixes)

These changes prevent the failure from recurring. Implement one or more based on your operational requirements.

### Fix 1: Dedicated EBS Volume for Logs (Recommended)

Isolate application logs from the root filesystem so OS stability is never at risk.

**Architecture:**

```
Root EBS volume (/dev/root)
├── Operating system
├── Nginx
└── Python application

Dedicated log EBS volume (/dev/nvme1n1)
└── /var/log/storage-breaker
```

**Steps:**

1. Create a new gp3 EBS volume in the same Availability Zone as the EC2 instance
2. Attach the volume to the instance
3. Identify the device:

```bash
lsblk
```

4. Format the new volume (only if empty):

```bash
sudo mkfs.ext4 /dev/nvme1n1
```

5. Stop the application:

```bash
sudo systemctl stop storage-breaker
```

6. Back up and recreate the mount point:

```bash
sudo mv /var/log/storage-breaker /var/log/storage-breaker.backup
sudo mkdir -p /var/log/storage-breaker
```

7. Mount and set permissions:

```bash
sudo mount /dev/nvme1n1 /var/log/storage-breaker
sudo chown -R ubuntu:ubuntu /var/log/storage-breaker
sudo chmod 750 /var/log/storage-breaker
```

8. Configure automatic mount in `/etc/fstab`:

```bash
sudo blkid /dev/nvme1n1
# Note the UUID

sudo cp /etc/fstab /etc/fstab.backup
sudo nano /etc/fstab
```

Add:

```
UUID=<your-uuid> /var/log/storage-breaker ext4 defaults,nofail 0 2
```

9. Test the fstab entry:

```bash
sudo umount /var/log/storage-breaker
sudo mount -a
df -h /var/log/storage-breaker
```

10. Start the application and verify:

```bash
sudo systemctl start storage-breaker
curl -i http://EC2_PUBLIC_IP/health
```

11. Clean up the backup after confirming everything works:

```bash
sudo rm -rf /var/log/storage-breaker.backup
```

**Benefit:** Application log growth cannot affect the operating system, Nginx, or SSH access.

---

### Fix 2: Increase Root EBS Volume

When more total storage is needed on the root filesystem.

**Steps:**

1. Modify the volume size in AWS Console: `EC2 → Volumes → Actions → Modify volume`
2. Extend the partition on the instance:

```bash
sudo apt install -y cloud-guest-utils
sudo growpart /dev/nvme0n1 1
```

3. Extend the filesystem:

```bash
# ext4
sudo resize2fs /dev/nvme0n1p1

# XFS
sudo xfs_growfs -d /
```

4. Verify:

```bash
df -h /
```

**Caveat:** This delays exhaustion but does not prevent it. Use alongside log rotation or a dedicated volume.

---

### Fix 3: Configure Log Rotation

For normal production logging where logs should not grow indefinitely.

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
| `size 100M` | Rotate when file reaches 100 MB |
| `rotate 7` | Keep 7 rotated files |
| `compress` | Compress old logs (saves space) |
| `delaycompress` | Keep most recent rotated file uncompressed |
| `copytruncate` | Copy-then-truncate so the app keeps writing to the same path |

Test:

```bash
sudo logrotate -d /etc/logrotate.d/storage-breaker
sudo logrotate -f /etc/logrotate.d/storage-breaker
ls -lh /var/log/storage-breaker
```

**Note:** For this specific drill, log rotation is intentionally disabled because the application is designed to fill the disk. Use log rotation only in production scenarios.

---

### Fix 4: Application-Level Disk Threshold

Add a safety check inside the application to stop writing before the disk is full.

```python
import shutil
from pathlib import Path

LOG_DIRECTORY = Path("/var/log/storage-breaker")
STOP_WRITING_THRESHOLD_PERCENT = 92.0
HEALTH_FAILURE_THRESHOLD_PERCENT = 90.0


def get_disk_usage_percent() -> float:
    usage = shutil.disk_usage(LOG_DIRECTORY)
    if usage.total == 0:
        return 0.0
    return (usage.used / usage.total) * 100


def should_stop_writing() -> bool:
    return get_disk_usage_percent() >= STOP_WRITING_THRESHOLD_PERCENT
```

**Threshold design:**

```
Disk usage < 90%  → healthy, writing logs
90% ≤ usage < 92% → unhealthy (health check returns 503), still writing
usage ≥ 92%       → stop writing, preserve OS stability
```

**Benefit:** The application proactively protects the OS, giving operators time to intervene before a full outage.

---

### Fix 5: systemd Journal Size Limits

Protect the system journal from unbounded growth (does not control application-specific log files):

```bash
sudo nano /etc/systemd/journald.conf
```

```ini
[Journal]
SystemMaxUse=500M
SystemKeepFree=1G
```

```bash
sudo systemctl restart systemd-journald
sudo journalctl --disk-usage
```

---

## Decision Matrix

| Scenario | Recommended Action |
| -------- | ------------------ |
| Disk is already full, need immediate recovery | **Temporary Resolution** — stop, truncate, restart |
| Production application with normal logging | **Fix 3** — log rotation |
| Application intentionally generates large logs | **Fix 1** — dedicated EBS volume |
| Need more room on root filesystem | **Fix 2** — increase root EBS |
| Want to prevent exhaustion at the source | **Fix 4** — application-level threshold |
| Protecting system journal | **Fix 5** — journald limits |
| Full production hardening | **Fix 1 + Fix 3 + Fix 4** combined |

---

## Professional SRE Considerations

### 1. Monitoring and Alerting

A reactive approach (manual `df -h`) is insufficient for production. Implement proactive monitoring.

**CloudWatch Metrics:**

```bash
# Install CloudWatch agent (if not present)
sudo apt install -y amazon-cloudwatch-agent

# Monitor disk utilization
aws cloudwatch put-metric-data \
    --namespace "StorageBreaker" \
    --metric-name "DiskUtilization" \
    --value 28 \
    --unit Percent \
    --dimensions InstanceId=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
```

**CloudWatch Alarms:**

| Alarm | Threshold | Action |
| ----- | --------- | ------ |
| DiskUtilization ≥ 80% | Warning | SNS notification |
| DiskUtilization ≥ 90% | Critical | Page on-call engineer |
| DiskUtilization ≥ 95% | Emergency | Auto-recovery or escalation |
| HealthCheckStatus = unhealthy | Immediate | Page on-call engineer |

**Custom Health Check Script (cron):**

```bash
#!/bin/bash
# /usr/local/bin/disk-check.sh

THRESHOLD=80
USAGE=$(df / --output=pcent | tail -1 | tr -d ' %')

if [ "$USAGE" -ge "$THRESHOLD" ]; then
    echo "DISK WARNING: Root filesystem at ${USAGE}%" | \
        logger -t disk-check
fi
```

```bash
# Run every 5 minutes
*/5 * * * * /usr/local/bin/disk-check.sh
```

### 2. Incident Response Runbook

Every production service should have a runbook. For this application:

**Runbook: Storage Breaker 503 Response**

```
1. CONFIRM the failure:
   curl -i http://<HOST>/health
   → 503 = unhealthy

2. CHECK disk usage:
   df -h /
   df -h /var/log/storage-breaker

3. IF disk ≥ 95%:
   a. sudo systemctl stop storage-breaker
   b. sudo truncate -s 0 /var/log/storage-breaker/application.log
   c. df -h /   → confirm space recovered
   d. sudo systemctl start storage-breaker
   e. curl -i http://<HOST>/health → confirm 200

4. IF disk < 95% but unhealthy:
   a. sudo systemctl status storage-breaker
   b. sudo journalctl -u storage-breaker -n 50
   c. Check for other issues (permissions, port conflicts)

5. ESCALATE if:
   - Disk keeps filling after recovery
   - Service won't start
   - Root filesystem is completely full (100%)
```

### 3. Post-Incident Review (Blameless Post-Mortem)

After any production incident, conduct a blameless post-mortem:

**Template:**

```markdown
# Post-Mortem: [Incident Title]

## Summary
- Date/Time:
- Duration:
- Impact:
- Root Cause:

## Timeline
- HH:MM — Event observed
- HH:MM — Investigation started
- HH:MM — Root cause identified
- HH:MM — Service restored

## What Went Well
-

## What Went Wrong
-

## Action Items
| # | Action | Owner | Due Date | Status |
|---|--------|-------|----------|--------|
| 1 |        |       |          |        |

## Lessons Learned
-
```

### 4. Capacity Planning

Track disk usage trends to predict when action is needed:

| Metric | Warning | Critical | Action |
| ------ | ------- | -------- | ------ |
| Disk usage | ≥ 70% | ≥ 85% | Plan volume expansion or log rotation |
| Log growth rate | > 1 GB/day | > 5 GB/day | Review log verbosity or add rotation |
| Days until full | < 7 days | < 2 days | Immediate intervention |

### 5. Change Management

Any infrastructure change should follow a process:

1. **Propose** — Document the change and its rationale
2. **Review** — Have another engineer review the plan
3. **Test** — Validate in a non-production environment
4. **Approve** — Get explicit approval before proceeding
5. **Execute** — Implement during a maintenance window
6. **Verify** — Confirm the change achieved the desired outcome
7. **Rollback plan** — Know how to undo the change if something goes wrong

### 6. Observability Stack

For production, consider a full observability stack:

| Layer | Tool | Purpose |
| ----- | ---- | ------- |
| Metrics | CloudWatch, Prometheus + Grafana | Disk usage, CPU, memory, request rate |
| Logs | CloudWatch Logs, ELK, Loki | Centralized log aggregation and search |
| Traces | X-Ray, Jaeger | Request tracing across services |
| Uptime | CloudWatch Synthetics, Pingdom | External health check monitoring |

### 7. Automation and Self-Healing

| Automation | Implementation |
| ---------- | -------------- |
| Auto-restart on failure | `systemd` `Restart=always` (already configured) |
| Disk cleanup cron job | Delete logs older than N days |
| Auto-scaling | Launch new instance if health check fails |
| Auto-recovery | AWS CloudWatch Recovery Actions |

**Cron-based cleanup example:**

```bash
#!/bin/bash
# /usr/local/bin/cleanup-logs.sh

find /var/log/storage-breaker -name "*.log.*" -mtime +7 -delete
find /var/log/storage-breaker -name "*.log" -size +500M -exec truncate -s 100M {} \;
```

### 8. Documentation Standards

Every incident and resolution should be documented with:

- **Commands used** with their exact output
- **Before/after state** (disk usage, service status)
- **Timeline** of actions taken
- **Decision rationale** — why this approach was chosen over alternatives
- **Verification** — proof that the fix worked

### 9. Security Considerations

| Concern | Mitigation |
| ------- | ---------- |
| Log file permissions | `640` — owner read/write, group read, no world access |
| Log directory permissions | `750` — no world access |
| Service runs as non-root | `User=ubuntu` in systemd unit |
| No sudo in application | Application never needs root |
| Volume encryption | Enable EBS encryption for log volume |

### 10. Testing the Recovery Process

Regularly test your recovery procedures:

1. **Schedule drills** — Monthly or quarterly chaos engineering exercises
2. **Use a staging environment** — Test changes before production
3. **Time the recovery** — Track mean time to recovery (MTTR)
4. **Rotate on-call** — Ensure multiple engineers can execute the runbook

---

## Recovery Verification Checklist

- [ ] Service stopped cleanly (`pgrep -af uvicorn` returns nothing)
- [ ] Disk space reclaimed (`df -h /` shows available space)
- [ ] No deleted-but-open files (`sudo lsof +L1` is clean)
- [ ] Application restarted (`systemctl status storage-breaker` shows active)
- [ ] Health endpoint returns `200 OK`
- [ ] Exactly one Uvicorn worker running
- [ ] Log growth is being monitored
- [ ] Root cause documented
- [ ] Long-term fix identified and scheduled
- [ ] Post-mortem completed (if production incident)
