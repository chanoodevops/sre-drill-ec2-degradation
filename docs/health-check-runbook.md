# Runbook: Health Check Failure RCA & Recovery

## Failure Summary

| Field | Detail |
| ----- | ------ |
| **Incident** | Health check returns `503 Service Unavailable` |
| **Common Root Causes** | Disk exhaustion, memory exhaustion, service crash, port conflict, Nginx misconfiguration, load balancer issues |
| **Impact** | Application unhealthy, users receive 503 errors |
| **Severity** | High — service degraded, potential cascade to SSH/Nginx/systemd failures |
| **Affected Component** | Storage Breaker application on EC2 instance |

---

## Quick Reference

### Troubleshooting Commands

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

### 503 Causes

| Cause | Check | Fix |
|-------|-------|-----|
| Disk full | `df -h /` | Truncate logs, add rotation |
| Memory exhausted | `free -h`, `dmesg \| grep oom` | Increase instance size, fix leak |
| Service crashed | `systemctl status`, `journalctl -u` | Restart, fix crash cause |
| Port conflict | `ss -lntp \| grep :3000` | Kill conflicting process |
| App bug | `curl localhost:3000/health` | Check app logs, fix code |
| Nginx misconfigured | `nginx -t`, `curl localhost/health` | Fix Nginx config, restart |
| Target group unhealthy | `aws elbv2 describe-target-health` | Fix instance health |

---

## Troubleshooting & Root Cause Analysis

This section covers how to identify the problem, whether you have monitoring in place or not.

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

### SSH Connection Fails

If SSH times out or refuses the connection, the issue is at the OS or network level, not the application.

**Symptoms:**

```bash
ssh ubuntu@EC2_PUBLIC_IP
# Connection timed out
# or
# ssh: connect to host EC2_PUBLIC_IP port 22: Connection refused
```

**Diagnosis:**

1. Verify the instance is running in AWS Console (`EC2 → Instances`)
2. Check Security Group allows port 22 from your IP:
   ```bash
   # In AWS CLI
   aws ec2 describe-security-groups --group-ids sg-XXXXXXXX
   ```
3. Verify you're using the correct PEM key:
   ```bash
   ssh -i <correct-key>.pem ubuntu@EC2_PUBLIC_IP
   ```
4. If the instance is running but SSH fails, the OS may be hung due to disk exhaustion or OOM — check the instance's system log in AWS Console (`EC2 → Actions → Monitor and troubleshoot → Get system log`)

**Resolution:** If the OS is hung, you may need to stop/start the instance or use AWS Systems Manager Session Manager to access it.

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

If disk is the problem, proceed to **Step A: Disk Exhaustion RCA** for detailed investigation.
If memory is the problem, proceed to **Step B: Memory Exhaustion RCA**.
For other failure categories, see the **Failure Classification** table and follow the corresponding step.

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
              (look for error messages)

Then → Apply the appropriate fix → Restart → Verify
```

---

### Failure Category Classification

When the health check returns 503, classify the failure before attempting fixes.

| Symptom | Category | Next Step |
|---------|----------|-----------|
| `df -h` shows ≥ 95% disk | **Disk Exhaustion** | → Step A |
| `free -h` shows no available memory, OOM in logs | **Memory Exhaustion** | → Step B |
| `pgrep uvicorn` returns nothing | **Service Not Running** | → Step C |
| `ss -lntp` shows port 3000 not listening | **Port Not Bound** | → Step D |
| Port 3000 listening, but `curl localhost:3000` fails | **Application Error** | → Step E |
| Port 3000 works, but external curl fails | **Network/Nginx Issue** | → Step F |
| Health endpoint returns 200 but load balancer shows 503 | **Load Balancer/Target Group** | → Step G |

---

### Step A: Disk Exhaustion RCA

```bash
# How full is the disk?
df -h /

# What's consuming space? Drill down from root
sudo du -xhd1 / 2>/dev/null | sort -rh | head -10
# Example: /var is 8GB → drill into /var

sudo du -sh /var/*
# Example: /var/log is 7.8GB → drill into /var/log

sudo du -sh /var/log/*
# Example: /var/log/storage-breaker is 7.6GB → found it

# Check the specific file
sudo ls -lh /var/log/storage-breaker/application.log
# Example: -rw-r----- 1 ubuntu ubuntu 7.6G Jul 17 14:12 application.log

# Check if inodes are also exhausted
df -ih
```

**Root cause confirmed:** Application log file grew until disk was 100% full.

**Evidence to document:**
- `df -h` output showing 100%
- `du` drill-down showing the culprit file
- File size and growth rate

---

### Step B: Memory Exhaustion RCA

```bash
# Check memory usage
free -h

# Check for OOM kills
sudo journalctl -k | grep -i "out of memory"
dmesg | grep -i "oom"

# Check which process used the most memory
ps aux --sort=-%mem | head -10

# Check swap usage
swapon --show
```

**Root cause candidates:**
- Memory leak in application
- Too many workers/processes
- Insufficient instance size for workload

---

### Step C: Service Not Running RCA

```bash
# Why did the service stop?
sudo journalctl -u storage-breaker -n 50 --no-pager

# Common reasons in the logs:
# - "No space left on device" → disk exhaustion
# - "Address already in use" → port conflict
# - "ModuleNotFoundError" → missing dependency
# - "Permission denied" → file permission issue

# Check if systemd tried to restart
sudo systemctl status storage-breaker
# Look for: "Started", "Stopped", "Failed", restart count

# Check if there's a core dump
coredumpctl list
```

---

### Step D: Port Not Bound RCA

```bash
# What's on port 3000?
sudo ss -lntp | grep :3000
# Empty = nothing listening

# What's on port 80?
sudo ss -lntp | grep :80

# Check if another process took the port
sudo lsof -i :3000

# Check Nginx error log for upstream errors
sudo tail -20 /var/log/nginx/storage-breaker-error.log
# Common: "connect() failed (111: Connection refused) 
#          while connecting to upstream"
```

---

### Step E: Application Error RCA

```bash
# Test the app directly (bypass Nginx)
curl -i http://127.0.0.1:3000/health

# If it returns 503, check the app's own logs
sudo tail -50 /var/log/storage-breaker/application.log

# Check the health endpoint logic
# The app checks: disk usage ≥ 95% OR writer status is disk_full/write_failed

# Check the writer state
curl -s http://127.0.0.1:3000/health | python3 -m json.tool

# Check if there are Python errors
sudo journalctl -u storage-breaker -n 100 | grep -i "error\|traceback\|exception"
```

---

### Step F: Network/Nginx RCA

```bash
# Test Nginx directly
curl -i http://127.0.0.1/health

# If Nginx returns 502 (Bad Gateway):
# - Uvicorn is not running or not on port 3000
sudo ss -lntp | grep :3000

# If Nginx returns 504 (Gateway Timeout):
# - App is too slow to respond
# Check Nginx timeouts
sudo grep -r "proxy_read_timeout\|proxy_connect_timeout" /etc/nginx/

# If Nginx returns 503:
# - Nginx can't reach upstream
# Check Nginx config
sudo nginx -t
sudo cat /etc/nginx/sites-enabled/storage-breaker

# Check Nginx is running
sudo systemctl status nginx
```

---

### Step G: Load Balancer / Target Group RCA

```bash
# Check if target group health shows unhealthy
aws elbv2 describe-target-health \
    --target-group-arn arn:aws:elasticloadbalancing:REGION:ACCOUNT:targetgroup/NAME/ID

# Check if the instance is registered
aws ec2 describe-instance-status \
    --instance-ids i-XXXXXXXXXXXXXXXXX

# Check CloudWatch metrics for the ALB
aws cloudwatch get-metric-statistics \
    --namespace AWS/ApplicationELB \
    --metric-name TargetResponseTime \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 60 \
    --statistics Average \
    --dimensions Name=LoadBalancer,Value=app/NAME/ID
```

---

### Correlate the Timeline

Build a timeline of events to confirm the root cause.

```
TIME        EVENT                           EVIDENCE
────        ─────                           ────────
14:00       App deployed, healthy           df -h: 28% disk
14:15       Log file starts growing         ls -lh: 50MB
14:30       Log file reaches 2GB            ls -lh: 2GB
14:45       Log file reaches 5GB            df -h: 80%
14:50       Log file reaches 7GB            df -h: 95%
14:52       Health check returns 503        curl: 503 Service Unavailable
14:52       Writer status: disk_full        curl /health: {"status":"unhealthy"}
14:55       Operator SSH'd in               SSH log
14:58       Truncated log, restarted        df -h: 28%
15:00       Health check returns 200        curl: 200 OK
```

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

Use this when the root filesystem is already at or near 100%. The goal is to restore service as quickly as possible. These steps apply to disk exhaustion — the most common cause of health check failures.

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

Choose the appropriate option based on whether you need to preserve the log data.

---

**Option A — Truncate (logs are not important, fastest recovery):**

```bash
sudo truncate -s 0 /var/log/storage-breaker/application.log
```

**Option B — Delete and recreate (application is stopped, logs not needed):**

```bash
sudo rm -f /var/log/storage-breaker/application.log
sudo touch /var/log/storage-breaker/application.log
sudo chown ubuntu:ubuntu /var/log/storage-breaker/application.log
sudo chmod 640 /var/log/storage-breaker/application.log
```

---

**Option C — Preserve logs: Copy to S3 before truncating (recommended for important logs):**

Use when logs contain audit data, compliance records, or debugging information you need later.

```bash
# 1. Stop the application
sudo systemctl stop storage-breaker

# 2. Install AWS CLI if not present
sudo apt install -y awscli

# 3. Upload the log to S3
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
aws s3 cp /var/log/storage-breaker/application.log \
    s3://YOUR-BACKUP-BUCKET/storage-breaker/${TIMESTAMP}/application.log

# 4. Verify upload completed
aws s3 ls s3://YOUR-BACKUP-BUCKET/storage-breaker/${TIMESTAMP}/

# 5. Truncate the local log
sudo truncate -s 0 /var/log/storage-breaker/application.log

# 6. Restart the service
sudo systemctl start storage-breaker
```

---

**Option D — Preserve logs: Compress in place (no S3, fastest local option):**

```bash
# 1. Stop the application
sudo systemctl stop storage-breaker

# 2. Compress the log file
sudo gzip -9 /var/log/storage-breaker/application.log
# Creates: application.log.gz

# 3. Check space freed
df -h /

# 4. Create a fresh empty log file
sudo touch /var/log/storage-breaker/application.log
sudo chown ubuntu:ubuntu /var/log/storage-breaker/application.log
sudo chmod 640 /var/log/storage-breaker/application.log

# 5. Restart
sudo systemctl start storage-breaker
```

Example result:

```
/var/log/storage-breaker/
├── application.log        (0 bytes — new, active log)
└── application.log.gz     (compressed — old log preserved)
```

Compression ratio for JSON logs is typically **80-90%**. An 8 GB log file may compress to ~800 MB.

---

**Option E — Preserve logs: Move to a different volume (quick EBS attach):**

Use when you need to preserve logs AND free root disk space simultaneously, and the log is too large for S3 upload or compression.

```bash
# 1. Attach a new EBS volume (done via AWS Console)
#    Note the device name, e.g., /dev/nvme1n1

# 2. Stop the application
sudo systemctl stop storage-breaker

# 3. Format if new (skip if already formatted)
sudo mkfs.ext4 /dev/nvme1n1

# 4. Mount temporarily
sudo mkdir -p /mnt/log-rescue
sudo mount /dev/nvme1n1 /mnt/log-rescue

# 5. Move the log to the new volume
sudo mv /var/log/storage-breaker/application.log \
    /mnt/log-rescue/application.log

# 6. Create fresh log file
sudo touch /var/log/storage-breaker/application.log
sudo chown ubuntu:ubuntu /var/log/storage-breaker/application.log
sudo chmod 640 /var/log/storage-breaker/application.log

# 7. Restart service
sudo systemctl start storage-breaker

# 8. Verify
df -h /
df -h /mnt/log-rescue

# 9. Upload to S3 later when disk pressure is gone
aws s3 cp /mnt/log-rescue/application.log \
    s3://YOUR-BACKUP-Bucket/storage-breaker/

# 10. Unmount and detach the rescue volume
sudo umount /mnt/log-rescue
# Then detach via AWS Console
```

---

**Option F — Preserve logs: Stream to remote via `rsync` (if SSH to another host is available):**

```bash
# 1. Stop the application
sudo systemctl stop storage-breaker

# 2. Copy log to remote server
rsync -avz /var/log/storage-breaker/application.log \
    backup-user@REMOTE_HOST:/backups/storage-breaker/

# 3. Truncate local log
sudo truncate -s 0 /var/log/storage-breaker/application.log

# 4. Restart
sudo systemctl start storage-breaker
```

---

### When to Use Each Option

| Scenario | Option | Why |
| -------- | ------ | --- |
| Logs not needed, speed is priority | **A** or **B** | Fastest recovery |
| Logs important, S3 available | **C** | Durable, versioned, cheap storage |
| Logs important, no S3, fast fix | **D** | Compression is instant, no network |
| Logs very large, need full copy | **E** | Move off the root volume entirely |
| Another server available | **F** | Simple network copy |

### Decision Flowchart

```
Are the logs important?
    │
    ├── NO → Option A (truncate) or B (delete)
    │
    └── YES → Is S3 available?
                │
                ├── YES → Option C (copy to S3, then truncate)
                │
                └── NO → Can you compress?
                            │
                            ├── YES → Option D (gzip in place)
                            │
                            └── NO → Another host available?
                                        │
                                        ├── YES → Option F (rsync)
                                        │
                                        └── NO → Option E (attach rescue EBS)
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

## Enterprise Prevention: Real-Time Log Shipping

The best emergency recovery is the one you never need. In production, logs should be streamed off the instance in real-time so local disk never fills up.

### Architecture Overview

```
EC2 Instance (local disk = temporary buffer only)
    │
    ├── Application writes to /var/log/storage-breaker/application.log
    │
    └── Log Shipper (Fluent Bit / CloudWatch Agent)
            │
            ├──→ CloudWatch Logs    (searchable, alerting, 90-day retention)
            ├──→ S3 Bucket          (long-term, lifecycle policies)
            └──→ Elasticsearch/Datadog/Splunk (if applicable)
```

### Option 1: Fluent Bit → CloudWatch Logs (AWS Native)

**Fluent Bit** is a lightweight log shipper (uses ~450 KB memory). AWS provides a managed version.

**Step 1: Install Fluent Bit**

```bash
# Add Fluent Bit repository
curl https://packages.fluentbit.io/fluentbit.key | gpg --dearmor > \
    /usr/share/keyrings/fluentbit-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/fluentbit-keyring.gpg] \
    https://packages.fluentbit.io/ubuntu/jammy jammy main" | \
    sudo tee /etc/apt/sources.list.d/fluentbit.list

sudo apt update
sudo apt install -y fluent-bit
```

**Step 2: Configure Fluent Bit**

```bash
sudo nano /etc/fluent-bit/fluent-bit.conf
```

```ini
[SERVICE]
    Flush        5
    Log_Level    info
    Daemon       off
    Parsers_File parsers.conf

[INPUT]
    Name              tail
    Path              /var/log/storage-breaker/application.log
    Parser            json
    Tag               storage-breaker
    Refresh_Interval  5
    Rotate_Wait       30
    Mem_Buf_Limit     10MB

[OUTPUT]
    Name                cloudwatch_logs
    Match               storage-breaker
    region              us-east-1
    log_group_name      /ec2/storage-breaker
    log_stream_name     ${HOSTNAME}
    auto_create_group   true
    retry_Limit         5
```

**Step 3: Create parsers file**

```bash
sudo nano /etc/fluent-bit/parsers.conf
```

Add:

```ini
[PARSER]
    Name        json
    Format      json
    Time_Key    timestamp
    Time_Format %Y-%m-%dT%H:%M:%S%z
```

**Step 4: Enable and start**

```bash
sudo systemctl enable fluent-bit
sudo systemctl start fluent-bit
```

**Step 5: Verify logs are shipping**

```bash
# Check Fluent Bit is running
sudo systemctl status fluent-bit

# Check CloudWatch Logs
aws logs describe-log-streams \
    --log-group-name /ec2/storage-breaker \
    --order-by LastEventTime \
    --descending \
    --limit 1

# Tail the log stream
aws logs tail /ec2/storage-breaker --follow
```

**Step 6: Set CloudWatch log retention**

```bash
aws logs put-retention-policy \
    --log-group-name /ec2/storage-breaker \
    --retention-in-days 90
```

---

### Option 2: CloudWatch Agent → CloudWatch Logs

AWS's own agent. Heavier than Fluent Bit but integrates natively with CloudWatch metrics.

**Step 1: Install CloudWatch Agent**

```bash
sudo apt install -y amazon-cloudwatch-agent
```

**Step 2: Create config**

```bash
sudo nano /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

```json
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/storage-breaker/application.log",
            "log_group_name": "/ec2/storage-breaker",
            "log_stream_name": "{instance_id}",
            "timezone": "UTC"
          }
        ]
      }
    },
    "log_stream_name": "storage-breaker-stream"
  },
  "metrics": {
    "namespace": "StorageBreaker",
    "metrics_collected": {
      "disk": {
        "measurement": ["used_percent"],
        "resources": ["/"]
      }
    }
  }
}
```

**Step 3: Start the agent**

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a fetch-config \
    -m ec2 \
    -s \
    -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

**Step 4: Verify**

```bash
# Check agent status
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a status

# Check logs in CloudWatch
aws logs tail /ec2/storage-breaker --follow
```

---

### Option 3: Fluent Bit → S3 (Direct to S3)

When you want logs in S3 without CloudWatch Logs as an intermediary.

**Step 1: Create S3 bucket with lifecycle**

```bash
# Create bucket
aws s3 mb s3://my-app-logs-$(aws sts get-caller-identity --query Account --output text)

# Set lifecycle policy (auto-delete after 365 days, transition to IA after 30 days)
aws s3api put-bucket-lifecycle-configuration \
    --bucket my-app-logs-ACCOUNT-ID \
    --lifecycle-configuration '{
        "Rules": [{
            "ID": "LogRetention",
            "Status": "Enabled",
            "Filter": {"Prefix": "storage-breaker/"},
            "Transitions": [
                {"Days": 30, "StorageClass": "STANDARD_IA"},
                {"Days": 90, "StorageClass": "GLACIER"}
            ],
            "Expiration": {"Days": 365}
        }]
    }'
```

**Step 2: Configure Fluent Bit for S3**

```ini
[OUTPUT]
    Name            s3
    Match           storage-breaker
    region          us-east-1
    bucket          my-app-logs-ACCOUNT-ID
    s3_key_format   /storage-breaker/$TAG/%Y/%m/%d/$UUID.json
    total_file_size 50M
    upload_timeout  5m
    use_put         true
    compression     gzip
```

---

### Option 4: Filebeat → ELK Stack

When using Elasticsearch, Logstash, Kibana.

**Step 1: Install Filebeat**

```bash
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.14.0-amd64.deb
sudo dpkg -i filebeat-8.14.0-amd64.deb
```

**Step 2: Configure Filebeat**

```bash
sudo nano /etc/filebeat/filebeat.yml
```

```yaml
filebeat.inputs:
  - type: log
    paths:
      - /var/log/storage-breaker/application.log
    json.keys_under_root: true
    json.add_error_key: true

output.elasticsearch:
  hosts: ["ELASTICSEARCH_HOST:9200"]
  index: "storage-breaker-%{+yyyy.MM.dd}"

setup.template.name: "storage-breaker"
setup.template.pattern: "storage-breaker-*"
setup.ilm.enabled: false
```

**Step 3: Enable and start**

```bash
sudo systemctl enable filebeat
sudo systemctl start filebeat
```

---

### Log Shipping Comparison

| Solution | Memory | Complexity | Best For |
|----------|--------|------------|----------|
| **Fluent Bit → CloudWatch** | ~450 KB | Low | AWS-native, lightweight |
| **CloudWatch Agent** | ~60 MB | Medium | AWS-native, need metrics too |
| **Fluent Bit → S3** | ~450 KB | Medium | Direct to S3, no CloudWatch |
| **Filebeat → ELK** | ~100 MB | High | Already running ELK stack |
| **rsync cron** | Negligible | Very Low | Simple, low-volume logging |

---

### With Log Shipping: Disk Never Fills

```
Normal flow:
    App writes → local log file → Fluent Bit tails → ships to S3/CloudWatch
                                         │
                              Local file can be rotated/truncated
                              because data is already shipped

If disk fills anyway:
    1. Stop app
    2. Truncate local log (data is safe in S3/CloudWatch)
    3. Restart app
    4. No data loss
```

---

### Combining With Other Fixes

| Fix | Without Log Shipping | With Log Shipping |
|-----|---------------------|-------------------|
| **Dedicated EBS** | Still needed to isolate logs | Nice to have, not critical |
| **Log rotation** | Required to cap local size | Can use aggressive rotation (rotate daily) |
| **Disk threshold** | Essential safety net | Safety net, but less likely to trigger |
| **S3 copy (Option C)** | Manual emergency step | Automatic — logs go to S3 in real-time |

**Recommended production stack:**

```
Fluent Bit → CloudWatch Logs (alerting + search)
           → S3 (long-term retention)
           + logrotate (cap local size at 100MB)
           + dedicated EBS volume (isolate from root)
```

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
