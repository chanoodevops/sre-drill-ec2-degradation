# Enterprise Incident Report: Disk Exhaustion — Detection, Recovery & Prevention

| Field | Value |
|-------|-------|
| **Document Type** | Enterprise Technical Report |
| **Version** | 1.0 |
| **Effective Date** | July 2026 |
| **Classification** | Internal — SRE / Infrastructure Teams |
| **Owner** | Site Reliability Engineering |
| **Review Cycle** | Quarterly |

---

## 1. Executive Summary

### 1.1 Purpose

This report establishes standardized procedures for detecting, recovering from, and preventing disk exhaustion incidents on production EC2 instances. It provides enterprise-grade guidance for maintaining service availability and data integrity.

### 1.2 Business Impact

Disk exhaustion represents a critical infrastructure failure with the following potential impacts:

| Impact Category | Severity | Business Consequence |
|-----------------|----------|---------------------|
| Service Availability | High | Application returns HTTP 503; users unable to access service |
| Data Integrity | High | Log data loss; loss of forensic evidence for incident analysis |
| Operational Stability | Medium | Cascading failures affecting SSH, Nginx, systemd |
| Compliance | Medium | Potential audit findings if logs are not preserved |

### 1.3 Recommended Solutions Summary

| Phase | Primary Solution | Fallback Solution | Cost | Data Preservation |
|-------|------------------|-------------------|------|-------------------|
| **Temporary Recovery** | S3 copy + gzip | Local gzip | ~$0.18 one-time | ✅ Yes |
| **Long-Term Prevention** | Log rotation + S3 shipping | App threshold + S3 shipping | ~$0.18/month | ✅ Yes |

**Note:** See Section 1.5 for detailed cost breakdown.

### 1.4 Key Findings

- Disk exhaustion typically provides 15–30 minutes of warning signs before complete failure
- Proactive detection reduces Mean Time to Recovery (MTTR) from hours to minutes
- **Data preservation requires S3 log shipping** — local-only solutions risk permanent data loss
- S3 storage costs approximately $0.023/GB/month, making data preservation cost-effective

### 1.5 Cost Analysis

**S3 Standard Pricing (us-east-1):**

| Component | Rate |
|-----------|------|
| Storage | $0.023 per GB per month |
| PUT requests | $0.005 per 1,000 requests |
| GET requests | $0.0004 per 1,000 requests |
| Data transfer | First 100 GB/month free |

**Monthly Cost by Log Size:**

| Log Size | S3 Storage | Requests (batched) | Total Cost |
|----------|------------|-------------------|------------|
| 1 GB | $0.023 | ~$0.0001 | **~$0.02** |
| 4 GB | $0.092 | ~$0.0004 | **~$0.09** |
| 8 GB | $0.184 | ~$0.0008 | **~$0.18** |
| 16 GB | $0.368 | ~$0.0016 | **~$0.37** |
| 50 GB | $1.150 | ~$0.005 | **~$1.16** |

**Note:** Fluent Bit batches logs into 50 MB chunks before uploading, reducing PUT request costs significantly.

---

## 2. Background

### 2.1 Definition

Disk exhaustion occurs when a filesystem reaches 100% capacity, preventing critical system operations including:

- Application log writes
- System journal logging
- Web server temporary file creation
- SSH session establishment
- Systemd service management

### 2.2 Impact Classification

| Severity Level | Disk Usage | System Impact | Required Response |
|----------------|------------|---------------|-------------------|
| **Info** | 70–80% | Normal operation | Monitor and log |
| **Warning** | 80–90% | Potential performance degradation | Investigate within 4 hours |
| **Critical** | 90–95% | Health checks may fail | Immediate intervention |
| **Emergency** | 95–100% | Service outage; system instability | Escalate within 15 minutes |

### 2.3 Root Cause Analysis

| Root Cause | Frequency | Prevention Strategy |
|------------|-----------|---------------------|
| Application log growth | High | Log rotation, S3 log shipping |
| System journal growth | Medium | Journald size limits |
| Temporary file accumulation | Low | Automated cleanup cron jobs |
| Core dump accumulation | Low | Coredumpctl configuration |
| Package cache growth | Low | Regular apt/yum cleanup |

---

## 3. Detection Procedures

### 3.1 Overview

When automated monitoring is unavailable, operators must perform manual Root Cause Analysis (RCA) following the structured approach outlined below.

### 3.2 Phase 1: Problem Confirmation (First 60 Seconds)

**Objective:** Verify the reported issue and assess initial system state.

```bash
# Step 1: Verify SSH connectivity
ssh -i key.pem ubuntu@EC2_PUBLIC_IP

# Step 2: Check health endpoint
curl -i http://EC2_PUBLIC_IP/health
# Expected responses:
# - 503 Service Unavailable = application unhealthy
# - Connection refused = service is down entirely

# Step 3: Check service status
sudo systemctl status storage-breaker
# Status indicators:
# - active (running) = service operational, issue may be external
# - inactive/failed = service down, requires immediate attention
```

### 3.3 Phase 2: System Triage (Next 2 Minutes)

**Objective:** Identify the failure category through systematic resource assessment.

```bash
# Execute diagnostic commands
df -h /          # Disk space assessment
free -h          # Memory utilization
uptime           # System load
pgrep -af uvicorn  # Process verification
```

**Diagnostic Interpretation:**

| Disk Status | Memory Status | Load Status | Likely Root Cause |
|-------------|---------------|-------------|-------------------|
| ≥ 95% usage | Normal | Normal | **Disk exhaustion** |
| Normal | Normal | Normal | Application defect or configuration issue |
| Normal | Normal | High load | CPU-bound process or infinite loop |
| Normal | High swap | Normal | Memory leak |

### 3.4 Phase 3: Disk Exhaustion Deep Dive

**Objective:** Identify the specific component consuming disk space.

```bash
# Step 1: Root-level disk analysis
sudo du -xhd1 / 2>/dev/null | sort -rh | head -10
```

**Example Output:**
```
8.2G    /var
2.4G    /usr
1.1G    /opt
```

**Step 2: Drill down into largest directory**

```bash
sudo du -sh /var/*
```

**Expected Output:**
```
8.0G    /var/log
```

**Step 3: Identify specific log directory**

```bash
sudo du -sh /var/log/*
```

**Expected Output:**
```
7.8G    /var/log/storage-breaker
```

**Step 4: Confirm file-level root cause**

```bash
sudo ls -lh /var/log/storage-breaker/application.log
```

**Expected Output:**
```
-rw-r----- 1 ubuntu ubuntu 7.8G Jul 17 14:12 application.log
```

**Root Cause Confirmed:** Application log file has consumed 7.8 GB of a 10 GB root volume.

### 3.5 Phase 4: Cascade Damage Assessment

**Objective:** Evaluate secondary system impacts before declaring resolution.

```bash
# Verify systemd journal functionality
sudo journalctl -n 5

# Check Nginx log write capability
sudo tail -5 /var/log/nginx/storage-breaker-error.log

# Identify deleted-but-open files holding disk space
sudo lsof +L1

# Assess inode exhaustion
df -ih
```

### 3.6 Phase 5: Evidence Documentation

**Objective:** Capture system state for post-incident analysis.

```bash
# Document current state
df -h > /tmp/rca-disk.log
sudo du -xhd1 / 2>/dev/null | sort -rh | head -10 > /tmp/rca-du.log
sudo ls -lh /var/log/storage-breaker/application.log > /tmp/rca-logfile.log
ps aux > /tmp/rca-processes.log
sudo ss -lntp > /tmp/rca-ports.log
sudo journalctl -u storage-breaker --since "30 min ago" --no-pager > /tmp/rca-journal.log
```

---

## 4. Emergency Recovery Procedures

### 4.1 Objective

Restore service availability within 5 minutes when disk usage is at or near 100%.

### 4.2 Pre-Recovery Checklist

Document the current system state before initiating recovery:

```bash
df -h > /tmp/pre-recovery-disk.log
sudo ls -lh /var/log/storage-breaker/application.log > /tmp/pre-recovery-log.log
ps aux > /tmp/pre-recovery-processes.log
sudo ss -lntp > /tmp/pre-recovery-ports.log
```

### 4.3 Primary Recovery Option: S3 Copy + Truncate

**Rationale:** Preserves log data durably in S3 while restoring local disk space.

**Cost:** One-time upload cost for the log file being recovered (e.g., 8 GB = ~$0.18 storage + minimal request fees).

**Prerequisites:**
- AWS CLI installed and configured
- S3 bucket exists with appropriate IAM permissions
- Network connectivity to AWS S3 endpoints

**Procedure:**

```bash
# Step 1: Stop application to prevent further writes
sudo systemctl stop storage-breaker
pgrep -af uvicorn  # Verify: no output expected

# Step 2: Document pre-recovery state
df -h > /tmp/pre-recovery-disk.log
sudo ls -lh /var/log/storage-breaker/application.log > /tmp/pre-recovery-log.log

# Step 3: Upload log to S3
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
aws s3 cp /var/log/storage-breaker/application.log \
    s3://your-backup-bucket/storage-breaker/${TIMESTAMP}/application.log

# Step 4: Verify upload completion
aws s3 ls s3://your-backup-bucket/storage-breaker/${TIMESTAMP}/

# Step 5: Truncate local log file
sudo truncate -s 0 /var/log/storage-breaker/application.log

# Step 6: Verify disk space recovery
df -h /

# Step 7: Check for deleted-but-open files
sudo lsof +L1

# Step 8: Restart application
sudo systemctl start storage-breaker

# Step 9: Verify service health
curl -i http://EC2_PUBLIC_IP/health
# Expected: HTTP/1.1 200 OK
```

### 4.4 Fallback Recovery Option: Local Gzip

**Rationale:** Preserves data locally when S3 is unavailable.

**Use Case:** S3 connectivity issues or temporary network outages.

```bash
# Step 1: Stop application
sudo systemctl stop storage-breaker

# Step 2: Compress log file (preserves data, frees 80-90% space)
sudo gzip -9 /var/log/storage-breaker/application.log
# Creates: application.log.gz

# Step 3: Create fresh empty log file
sudo touch /var/log/storage-breaker/application.log
sudo chown ubuntu:ubuntu /var/log/storage-breaker/application.log
sudo chmod 640 /var/log/storage-breaker/application.log

# Step 4: Verify disk space recovery
df -h /

# Step 5: Restart and verify
sudo systemctl start storage-breaker
curl -i http://EC2_PUBLIC_IP/health
```

**Expected Result:**
```
/var/log/storage-breaker/
├── application.log        (0 bytes — new, active log)
└── application.log.gz     (compressed — old log preserved)
```

**Trade-off:** Recovery takes 10-30 seconds depending on file size, but data is preserved locally for analysis.

### 4.5 Post-Recovery Verification

- [ ] Disk space reclaimed (`df -h /` shows available space)
- [ ] No deleted-but-open files (`sudo lsof +L1` is clean)
- [ ] Application restarted (`systemctl status storage-breaker` shows active)
- [ ] Health endpoint returns `200 OK`
- [ ] Exactly one Uvicorn worker running
- [ ] Service stable for 5 minutes

---

## 5. Long-Term Prevention

### 5.1 Objective

Eliminate the possibility of disk exhaustion causing service outages while ensuring complete data preservation.

### 5.2 Data Preservation Strategy

```
Application → writes to local log → Fluent Bit tails → ships to S3 (real-time)
                                    ↓
                          Local rotation is safe
                          because data is already in S3
```

**Key Principle:** Data preservation requires log shipping to S3. Local rotation protects disk capacity; S3 protects data integrity.

### 5.3 Primary Solution: Log Rotation + S3 Log Shipping

**Rationale:** Combines automatic disk protection with durable data preservation.

**Cost-Benefit Analysis (8 GB log example):**

| Component | Calculation | Monthly Cost |
|-----------|-------------|--------------|
| S3 Storage | 8 GB × $0.023/GB | $0.184 |
| PUT Requests | ~160 requests (50 MB batches) × $0.005/1,000 | $0.0008 |
| Data Transfer | < 100 GB/month | $0.00 (free) |
| **Total** | | **~$0.18** |

| Metric | Value |
|--------|-------|
| Fluent Bit Resource Usage | ~450 KB memory, negligible CPU |
| Local Storage Impact | Max 800 MB (7 × 100 MB rotated files) |
| Data Preservation | 100% — all logs permanently stored in S3 |

**Step 1: Install Fluent Bit Log Shipper**

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

**Step 2: Configure Fluent Bit for S3 Shipping**

```bash
sudo tee /etc/fluent-bit/fluent-bit.conf << 'EOF'
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
    Name                s3
    Match               storage-breaker
    region              us-east-1
    bucket              your-log-bucket
    s3_key_format       /storage-breaker/$TAG/%Y/%m/%d/$UUID.json
    total_file_size     50M
    upload_timeout      5m
    use_put             true
    compression         gzip
EOF
```

**Step 3: Create Parsers Configuration**

```bash
sudo tee /etc/fluent-bit/parsers.conf << 'EOF'
[PARSER]
    Name        json
    Format      json
    Time_Key    timestamp
    Time_Format %Y-%m-%dT%H:%M:%S%z
EOF
```

**Step 4: Enable and Start Fluent Bit**

```bash
sudo systemctl enable fluent-bit
sudo systemctl start fluent-bit
sudo systemctl status fluent-bit
```

**Step 5: Configure Log Rotation**

```bash
sudo tee /etc/logrotate.d/storage-breaker << 'EOF'
/var/log/storage-breaker/application.log {
    size 100M
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    create 0640 ubuntu ubuntu
    dateext
    dateformat -%Y%m%d
}
EOF
```

**Step 6: Verify Configuration**

```bash
# Verify Fluent Bit is shipping logs
sudo systemctl status fluent-bit

# Verify S3 contains logs
aws s3 ls s3://your-log-bucket/storage-breaker/ --recursive

# Verify log rotation configuration
sudo logrotate -d /etc/logrotate.d/storage-breaker
```

**Data Flow Diagram:**

```
Application
    │
    ├──→ Local log file (100 MB max, rotated daily)
    │
    └──→ Fluent Bit (real-time log shipper)
              │
              └──→ S3 Bucket (permanent storage, lifecycle policies)
```

### 5.4 Fallback Solution: Application Threshold + S3 Log Shipping

**Rationale:** Application-level protection when log rotation is not feasible.

**Use Cases:**
- Application requires continuous log file path
- Application does not support log rotation
- Legacy systems without logrotate access

**Implementation:**

```python
import shutil
from pathlib import Path

LOG_DIRECTORY = Path("/var/log/storage-breaker")
STOP_WRITING_THRESHOLD = 92.0
HEALTH_FAILURE_THRESHOLD = 90.0

def get_disk_usage_percent() -> float:
    usage = shutil.disk_usage(LOG_DIRECTORY)
    if usage.total == 0:
        return 0.0
    return (usage.used / usage.total) * 100

def should_stop_writing() -> bool:
    return get_disk_usage_percent() >= STOP_WRITING_THRESHOLD

def is_healthy() -> bool:
    return get_disk_usage_percent() < HEALTH_FAILURE_THRESHOLD
```

**Threshold Design:**

| Disk Usage | Application Behavior |
|------------|---------------------|
| < 90% | Healthy — writing logs normally |
| 90% – 92% | Unhealthy — returns HTTP 503, still writing |
| ≥ 92% | Critical — stops writing, preserves OS stability |

**Comparison: Log Rotation vs Application Threshold**

| Factor | Log Rotation + S3 | App Threshold + S3 |
|--------|-------------------|---------------------|
| Implementation Cost | $0 (5 minutes) | $0 (code change required) |
| Ongoing Cost | ~$0.18/month | ~$0.18/month |
| Data Preservation | ✅ All logs in S3 | ✅ All logs in S3 |
| Local Disk Usage | Max 800 MB | Max 92% of disk |
| Requires Application Changes | ❌ No | ✅ Yes |
| Prevents Disk Exhaustion | ✅ Yes (caps at 100MB) | ✅ Yes (stops at 92%) |

---

## 6. Decision Matrix

### 6.1 Recovery Decision Tree

| Scenario | Temporary Recovery | Long-Term Prevention | Total Cost | Data Safety |
|----------|-------------------|----------------------|------------|-------------|
| **Standard Production** | S3 copy + gzip | Log rotation + S3 shipping | ~$0.18/month | ✅ Full |
| **App Cannot Use Rotation** | S3 copy + gzip | App threshold + S3 shipping | ~$0.18/month | ✅ Full |
| **S3 Unavailable (Not Recommended)** | Local gzip | Log rotation only | $0 | ⚠️ Local only |
| **No S3, No Rotation (Not Recommended)** | Local gzip | App threshold only | $0 | ⚠️ Partial loss |

**Critical Note:** Data preservation requires S3 log shipping. Without S3, logs are only preserved locally and may be permanently lost during incidents.

---

## 7. Monitoring & Alerting Strategy

### 7.1 Alert Thresholds

| Alert Level | Disk Usage Threshold | Response Time | Required Action |
|-------------|---------------------|---------------|-----------------|
| **Info** | ≥ 70% | 24 hours | Log event, no immediate action |
| **Warning** | ≥ 80% | 4 hours | Notify on-call, investigate |
| **Critical** | ≥ 90% | 1 hour | Page on-call, immediate intervention |
| **Emergency** | ≥ 95% | 15 minutes | Auto-remediation or escalation |

### 7.2 Alert Channel Configuration

| Channel | Use Case | Integration Method |
|---------|----------|-------------------|
| Email | Non-urgent warnings | AWS SNS → Email |
| Slack | Team notifications | AWS SNS → Lambda → Slack Webhook |
| PagerDuty | Critical/Emergency | AWS SNS → PagerDuty Integration |
| SMS | After-hours emergencies | AWS SNS → SMS |

### 7.3 Runbook Integration

When disk alerts trigger:
1. Open the **Health Check Runbook** (`docs/health-check-runbook.md`)
2. Execute **Phase 1: Confirm the Problem**
3. Proceed to **Phase 2: Quick Triage**
4. Apply resolution based on **Decision Matrix** (Section 6)

---

## 8. Capacity Planning

### 8.1 Disk Usage Forecasting

| Metric | Warning Threshold | Critical Threshold | Required Action |
|--------|-------------------|-------------------|-----------------|
| Disk Usage | ≥ 70% | ≥ 85% | Plan volume expansion or implement rotation |
| Log Growth Rate | > 1 GB/day | > 5 GB/day | Review log verbosity, add rotation |
| Days Until Full | < 7 days | < 2 days | Immediate intervention required |

### 8.2 Growth Rate Measurement

```bash
# Measure log growth over 1 hour
SIZE_BEFORE=$(stat -c%s /var/log/storage-breaker/application.log)
sleep 3600
SIZE_AFTER=$(stat -c%s /var/log/storage-breaker/application.log)

GROWTH_PER_HOUR=$((SIZE_AFTER - SIZE_BEFORE))
GROWTH_PER_DAY=$((GROWTH_PER_HOUR * 24))

echo "Daily growth rate: $(numfmt --to=iec $GROWTH_PER_DAY)/day"
```

### 8.3 Capacity Recommendations

| Log Growth Rate | Root Volume Size | Dedicated Log Volume | Log Rotation Frequency |
|-----------------|------------------|----------------------|------------------------|
| < 100 MB/day | 30 GB | Optional | Daily |
| 100 MB – 1 GB/day | 30 GB | Recommended | Hourly |
| > 1 GB/day | 30 GB | Required | Hourly + S3 shipping |

---

## 9. Security Considerations

| Security Concern | Mitigation Strategy |
|------------------|---------------------|
| Log File Permissions | `640` — owner read/write, group read, no world access |
| Log Directory Permissions | `750` — no world access |
| Service Execution | Non-root user (`User=ubuntu` in systemd unit) |
| Application Privileges | No sudo required for application operations |
| Volume Encryption | Enable EBS encryption for log volumes |
| Log Retention | Configure S3 lifecycle policies for compliance |

---

## 10. Incident Response Process

### 10.1 Escalation Matrix

| Severity Level | Response Time | Escalation Path |
|----------------|---------------|-----------------|
| **P1 — Service Down** | 15 minutes | Immediately page on-call engineer |
| **P2 — Service Degraded** | 1 hour | Notify on-call via Slack |
| **P3 — Potential Issue** | 4 hours | Log ticket, investigate next business day |

### 10.2 Communication Template

```
Subject: [P1] Storage Breaker Health Check Failure — Disk Exhaustion

Impact: Health check returning HTTP 503; users unable to access service
Duration: [START_TIME] — [END_TIME] UTC
Root Cause: Disk exhaustion on /dev/root
Status: Investigating / Mitigating / Resolved

Next update: [TIME] UTC
```

---

## 11. Post-Incident Review

### 11.1 Blameless Post-Mortem Template

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

### 11.2 Key Performance Metrics

| Metric | Target | Current |
|--------|--------|---------|
| Mean Time to Detection (MTTD) | < 5 minutes | |
| Mean Time to Recovery (MTTR) | < 15 minutes | |
| Monthly Disk Exhaustion Incidents | 0 | |
| Alert Coverage | 100% of instances | |

---

## 12. References

| Document | Description |
|----------|-------------|
| [Deployment Guide](deployment.md) | Infrastructure setup and application deployment |
| [API Reference](api-reference.md) | Application endpoint documentation |
| [Systemd Guide](systemd-guide.md) | Service management, logging, and security |
| [Nginx Guide](nginx-guide.md) | Reverse proxy, HTTPS, Let's Encrypt, ACM |
| [Health Check Runbook](health-check-runbook.md) | Troubleshooting, RCA, and recovery procedures |

---

## Appendix A: Quick Reference Commands

| Task | Command |
|------|---------|
| Check disk usage | `df -h /` |
| Check inode usage | `df -ih` |
| Find largest directories | `sudo du -xhd1 / 2>/dev/null \| sort -rh \| head -10` |
| Check log file size | `ls -lh /var/log/storage-breaker/application.log` |
| Check for deleted open files | `sudo lsof +L1` |
| Test health endpoint | `curl -i http://127.0.0.1:3000/health` |
| Check service status | `sudo systemctl status storage-breaker` |
| View service logs | `sudo journalctl -u storage-breaker -n 50` |

---

## Appendix B: Alarm Configuration Examples

### AWS CLI — CloudWatch Alarm

```bash
aws cloudwatch put-metric-alarm \
    --alarm-name "DiskCritical-$(curl -s http://169.254.169.254/latest/meta-data/instance-id)" \
    --alarm-description "Root volume disk usage >= 90%" \
    --metric-name DiskSpaceUtilization \
    --namespace StorageBreaker \
    --statistic Average \
    --period 300 \
    --threshold 90 \
    --comparison-operator GreaterThanOrEqualToThreshold \
    --evaluation-periods 1 \
    --alarm-actions arn:aws:sns:us-east-1:ACCOUNT:disk-alerts \
    --dimensions Name=InstanceId,Value=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
```

### Terraform — CloudWatch Alarm

```hcl
resource "aws_cloudwatch_metric_alarm" "disk_critical" {
  alarm_name          = "DiskCritical-${aws_instance.app.id}"
  alarm_description   = "Root volume disk usage >= 90%"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 1
  metric_name         = "DiskSpaceUtilization"
  namespace           = "StorageBreaker"
  period              = 300
  statistic           = "Average"
  threshold           = 90
  alarm_actions       = [aws_sns_topic.disk_alerts.arn]

  dimensions = {
    InstanceId = aws_instance.app.id
  }
}
```

---

**Document Control**

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | July 2026 | SRE Team | Initial release |

---

**END OF DOCUMENT**
