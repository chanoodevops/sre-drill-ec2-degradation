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

## Real-World Detection: How Do We Know?

In production, you don't wait for a `curl` to tell you the disk is full. Here's how SREs detect disk exhaustion before it becomes an outage.

### 1. CloudWatch Metrics (AWS Native)

AWS automatically publishes EBS volume metrics to CloudWatch every 5 minutes.

**Key metrics to monitor:**

| Metric | Namespace | Description |
| ------ | --------- | ----------- |
| `DiskSpaceUtilization` | `CWAgent` | Percentage of disk used (requires CloudWatch agent) |
| `VolumeReadOps` | `AWS/EBS` | Read operations per second |
| `VolumeWriteOps` | `AWS/EBS` | Write operations per second |
| `BurstBalance` | `AWS/EBS` | gp2/gp3 burst credit balance |

**Install CloudWatch agent for disk metrics:**

```bash
# Download and install
sudo apt install -y amazon-cloudwatch-agent

# Create config
sudo nano /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

```json
{
  "metrics": {
    "namespace": "StorageBreaker",
    "metrics_collected": {
      "disk": {
        "measurement": ["used_percent", "inodes_free"],
        "resources": ["/"]
      }
    }
  }
}
```

```bash
# Start the agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a fetch-config \
    -m ec2 \
    -s \
    -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

**Create CloudWatch Alarm:**

```bash
aws cloudwatch put-metric-alarm \
    --alarm-name "DiskCritical-RootVolume" \
    --alarm-description "Root volume disk usage >= 85%" \
    --metric-name DiskSpaceUtilization \
    --namespace StorageBreaker \
    --statistic Average \
    --period 300 \
    --threshold 85 \
    --comparison-operator GreaterThanOrEqualToThreshold \
    --evaluation-periods 1 \
    --alarm-actions arn:aws:sns:us-east-1:ACCOUNT-ID:disk-alerts \
    --dimensions Name=InstanceId,Value=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
```

---

### 2. Prometheus + Grafana (Open Source)

**Node Exporter** exposes disk metrics as Prometheus scrape targets.

```bash
# Install Node Exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar xzf node_exporter-1.7.0.linux-amd64.tar.gz
sudo cp node_exporter-1.7.0.linux-amd64/node_exporter /usr/local/bin/

# Create systemd service
sudo nano /etc/systemd/system/node_exporter.service
```

```ini
[Unit]
Description=Prometheus Node Exporter
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/node_exporter
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
```

**Key Prometheus queries:**

```promql
# Disk usage percentage
(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100

# Disk usage rate (how fast it's filling)
deriv(node_filesystem_avail_bytes{mountpoint="/"}[1h]) * -1

# Time until full (hours remaining)
node_filesystem_avail_bytes{mountpoint="/"} / (deriv(node_filesystem_avail_bytes{mountpoint="/"}[1h]) * -1) / 3600
```

---

### 3. Syslog + Log-Based Alerts

Forward system logs to a centralized platform and alert on disk-related messages.

**Syslog messages to watch:**

```bash
# "No space left on device" errors
grep -i "no space left" /var/log/syslog

# Filesystem full warnings
grep -i "filesystem full" /var/log/kern.log

# Out-of-memory or journal failures
journalctl -p err --since "1 hour ago"
```

**ELK / Loki alert rules:**

```yaml
# Grafana Loki alert rule
- alert: DiskExhaustion
  expr: |
    sum by (instance) (rate({job="node"} |~ "No space left on device"[5m])) > 0
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Disk exhaustion detected on {{ $labels.instance }}"
```

---

### 4. Health Check Monitoring (External)

External uptime monitors detect when your health endpoint starts returning 503.

| Tool | How It Works |
| ---- | ------------ |
| **AWS CloudWatch Synthetics** | Canary scripts hit `/health` every 1 minute |
| **Pingdom / UptimeRobot** | HTTP checks from external locations |
| **Grafana OnCall** | Integrated alerting with escalation |
| **PagerDuty** | Incident management with on-call routing |

**CloudWatch Synthetics canary:**

```javascript
// canary.js
const https = require("https");

exports.handler = async () => {
    const url = "http://EC2_PUBLIC_IP/health";
    return new Promise((resolve, reject) => {
        https.get(url, (res) => {
            if (res.statusCode !== 200) {
                reject(new Error(`Health check returned ${res.statusCode}`));
            } else {
                resolve("OK");
            }
        }).on("error", reject);
    });
};
```

---

### 5. System-Level Detection (On-Instance)

**Cron-based disk monitoring script:**

```bash
#!/bin/bash
# /usr/local/bin/disk-monitor.sh

ALERT_THRESHOLD=80
CRITICAL_THRESHOLD=90
LOG_FILE="/var/log/disk-monitor.log"

USAGE=$(df / --output=pcent | tail -1 | tr -d ' %')
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

if [ "$USAGE" -ge "$CRITICAL_THRESHOLD" ]; then
    echo "[$TIMESTAMP] CRITICAL: Root filesystem at ${USAGE}%" >> "$LOG_FILE"
    # Send SNS notification
    aws sns publish \
        --topic-arn "arn:aws:sns:us-east-1:ACCOUNT:disk-alerts" \
        --message "CRITICAL: Disk usage at ${USAGE}% on $(hostname)" \
        --subject "Disk Critical"
elif [ "$USAGE" -ge "$ALERT_THRESHOLD" ]; then
    echo "[$TIMESTAMP] WARNING: Root filesystem at ${USAGE}%" >> "$LOG_FILE"
fi
```

```bash
# Add to crontab
echo "*/5 * * * * /usr/local/bin/disk-monitor.sh" | crontab -
```

**Systemd timer (modern alternative to cron):**

```ini
# /etc/systemd/system/disk-monitor.service
[Unit]
Description=Disk usage monitor

[Service]
Type=oneshot
ExecStart=/usr/local/bin/disk-monitor.sh
```

```ini
# /etc/systemd/system/disk-monitor.timer
[Unit]
Description=Run disk monitor every 5 minutes

[Timer]
OnBootSec=5min
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target
```

---

### 6. CloudWatch Agent + SNS Integration (End-to-End)

This is the most common production setup:

```
EC2 Instance
    │
    ├── CloudWatch Agent → publishes DiskSpaceUtilization metric
    │
    └── CloudWatch Alarm → triggers when usage >= 85%
            │
            └── SNS Topic → sends email / SMS / Slack
                    │
                    └── Lambda → auto-remediate (truncate logs, page on-call)
```

---

### Detection Method Comparison

| Method | Detection Speed | Setup Effort | Cost | Best For |
| ------ | --------------- | ------------ | ---- | -------- |
| `df -h` (manual) | Minutes | None | Free | Drills, small environments |
| CloudWatch Agent | 5 min intervals | Medium | Low | AWS-native setups |
| Prometheus + Grafana | 15s–1min | Medium | Free | Multi-cloud, Kubernetes |
| Syslog + ELK/Loki | Near real-time | High | Medium | Centralized log analysis |
| Health check monitor | 1 min intervals | Low | Free/Low | External availability |
| Cron script | 5 min intervals | Low | Free | Quick on-instance setup |

### Signal Cascade: What Happens as Disk Fills

```
Disk Usage    System Behavior                          Detection Signal
──────────    ─────────────────────────────────────    ──────────────────────
70-80%        Normal operation                         CloudWatch warning
80-85%        Logs may slow down                       CloudWatch alarm fires
85-90%        Nginx temp files may fail                Application errors
90-95%        Health check returns 503                 Uptime monitor alerts
95-100%       SSH may fail, OS unstable                PagerDuty escalation
100%          System hang, potential data loss         All alerts firing
```

---

## No Monitoring? Troubleshooting From Scratch

In many real-world scenarios, you get paged for a 503 and there's no dashboard, no alerts, no history. You're flying blind. Here's the systematic approach.

### Phase 1: First 60 Seconds — Confirm the Problem

You received an alert (or a user complaint). Before touching anything, confirm what's broken.

```bash
# 1. Can you even SSH in?
ssh -i key.pem ubuntu@EC2_PUBLIC_IP

# 2. Check the health endpoint
curl -i http://EC2_PUBLIC_IP/health
# 503 = something is wrong
# Connection refused = service is down entirely

# 3. Check if the service is running
sudo systemctl status storage-breaker
# active (running) = service is up, problem may be elsewhere
# inactive/failed = service is down
```

**At this point you know:**
- SSH works → OS is not fully hung
- 503 from health check → application sees a problem
- Service status → whether it's running or crashed

---

### Phase 2: Quick Triage — Find the Category

Run these 4 commands to narrow down the issue:

```bash
# 1. Disk space (most common for 503)
df -h /

# 2. Memory
free -h

# 3. CPU / load
uptime

# 4. Service process
pgrep -af uvicorn
```

**Interpret the results:**

| df -h shows | free -h shows | uptime shows | Likely cause |
|-------------|---------------|--------------|--------------|
| ≥ 95% disk | Normal | Normal | **Disk exhaustion** |
| Normal | Normal | Normal | Application bug or config issue |
| Normal | Normal | High load | CPU-bound or infinite loop |
| Normal | High swap | Normal | Memory leak |

---

### Phase 3: Deep Dive — Confirm Root Cause

If disk is the problem, investigate further:

```bash
# What's using the disk?
sudo du -xhd1 / 2>/dev/null | sort -rh | head -10
```

Example output:
```
8.2G    /var
2.4G    /usr
1.1G    /opt
```

→ `/var` is the problem. Drill down:

```bash
sudo du -sh /var/*
```

```
8.0G    /var/log
```

→ `/var/log` is the problem. Drill down:

```bash
sudo du -sh /var/log/*
```

```
7.8G    /var/log/storage-breaker
```

→ Found it. Check the file:

```bash
sudo ls -lh /var/log/storage-breaker/application.log
```

```
-rw-r----- 1 ubuntu ubuntu 7.8G Jul 17 14:12 application.log
```

**Root cause confirmed: Application log file consumed 7.8 GB of an 8.7 GB root volume.**

---

### Phase 4: Check for Cascade Damage

Disk exhaustion often breaks other things. Check before declaring victory:

```bash
# Can systemd write journal logs?
sudo journalctl -n 5

# Can Nginx write logs?
sudo tail -5 /var/log/nginx/storage-breaker-error.log

# Are there deleted-but-open files holding space?
sudo lsof +L1

# Are inodes exhausted too?
df -ih
```

---

### Phase 5: Decision — Fix or Mitigate?

```
Is the service still running?
    │
    ├── YES → Stop it first (stop the bleeding)
    │         sudo systemctl stop storage-breaker
    │
    └── NO → Check why it stopped
              sudo journalctl -u storage-breaker -n 50
              (may show "No space left on device" errors)

Then → Free disk space → Restart → Verify
```

---

### Quick Reference: Troubleshooting Commands

| Command | What it tells you |
|---------|-------------------|
| `df -h /` | Is disk full? |
| `df -ih` | Are inodes exhausted? |
| `free -h` | Is memory the issue? |
| `uptime` | Is the system overloaded? |
| `sudo du -xhd1 / \| sort -rh \| head -10` | What directory is largest? |
| `sudo ls -lh /var/log/storage-breaker/` | How big is the log file? |
| `pgrep -af uvicorn` | Is the app running? How many workers? |
| `sudo systemctl status storage-breaker` | Service state |
| `sudo journalctl -u storage-breaker -n 50` | Recent service logs |
| `sudo lsof +L1` | Deleted files still open |
| `sudo ss -lntp \| grep -E ':80\|:3000'` | Are ports listening? |
| `curl -i http://127.0.0.1:3000/health` | Health check from inside |

---

### Troubleshooting Flowchart

```
                    503 Health Check
                          │
                    ┌─────┴─────┐
                    │           │
              SSH works?    SSH fails?
                 │              │
            ┌────┴────┐    OS is hung or
            │         │    network issue
        Service     Service
        running?    crashed?
            │         │
       ┌────┴───┐    journalctl -u storage-breaker
       │        │    (check why it crashed)
   df -h     free -h
   disk?     memory?
       │        │
  ┌────┴───┐   ┌┴────────┐
  │        │   │          │
 ≥95%    Normal  High    Normal
  │       │     swap     │
 DU full  App    OOM     App bug
          bug    kill
```

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
