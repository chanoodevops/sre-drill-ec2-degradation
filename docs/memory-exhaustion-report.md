# Enterprise Incident Report: Memory Exhaustion — Detection, Recovery & Prevention

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

This report establishes standardized procedures for detecting, recovering from, and preventing memory exhaustion incidents on production EC2 instances running the Storage Breaker application. It provides enterprise-grade guidance for maintaining service availability and preventing OOM Killer-induced outages.

### 1.2 Business Impact

Memory exhaustion represents a critical infrastructure failure with the following potential impacts:

| Impact Category | Severity | Business Consequence |
|-----------------|----------|---------------------|
| Service Availability | High | Application returns HTTP 503 or 502; OOM Killer terminates Uvicorn worker |
| Operational Stability | High | Cascading service restarts; potential boot loops if OOM occurs at startup |
| Data Integrity | Medium | In-flight log writes may be interrupted; partial log records |
| System Stability | Medium | Kernel OOM Killer may terminate other critical processes (sshd, nginx) |

### 1.3 Recommended Solutions Summary

| Phase | Primary Solution | Fallback Solution | Cost | Service Impact |
|-------|------------------|-------------------|------|----------------|
| **Temporary Recovery** | Service restart + swap creation | cgroup memory limit + restart | ~$0 one-time | 30–60 second outage |
| **Long-Term Prevention** | Right-sizing + systemd memory guard | Application-level memory profiling | ~$15/month (instance upgrade) | Zero downtime |

**Note:** See Section 8 for detailed capacity planning.

### 1.4 Key Findings

- Memory exhaustion on t2.micro (1 GiB) instances occurs within minutes of sustained load
- The Linux OOM Killer terminates processes without warning — **no graceful shutdown occurs**
- Adding swap is a temporary buffer, not a solution — it masks the underlying capacity issue
- Right-sizing the instance to t3.small (2 GiB) is the only reliable long-term fix
- Systemd `MemoryMax` limits provide deterministic failure — the service restarts cleanly instead of the kernel OOM Killer choosing randomly

### 1.5 Cost Analysis

**Instance Right-Sizing (Monthly):**

| Instance | vCPUs | RAM | Monthly Cost | Recommended For |
|----------|-------|-----|-------------|-----------------|
| t2.micro | 1 | 1 GiB | ~$8.47 | ❌ Insufficient |
| t3.small | 2 | 2 GiB | ~$15.18 | ✅ Minimum recommended |
| t3.medium | 2 | 4 GiB | ~$30.37 | ✅ Production with headroom |

**Swap File Cost:**

| Swap Size | EBS Cost (gp3) | Performance Impact |
|-----------|----------------|-------------------|
| 2 GB | ~$0.16/month | Minimal if swappiness is low |
| 4 GB | ~$0.32/month | Moderate — disk thrashing risk |

---

## 2. Background

### 2.1 Definition

Memory exhaustion occurs when a system's available RAM is fully consumed, triggering the Linux OOM (Out of Memory) Killer to terminate processes to reclaim memory. On a t2.micro instance with 1 GiB RAM, the Uvicorn worker (typically 80–150 MB RSS baseline, growing under load) competes with:

- Operating system kernel and system services (~200 MB)
- Nginx reverse proxy (~10–30 MB)
- Systemd journal and logging (~20–50 MB)
- SSH daemon and cron (~10–20 MB)
- Python interpreter overhead (~30–50 MB)

### 2.2 Impact Classification

| Severity Level | Available Memory | System Impact | Required Response |
|----------------|-----------------|---------------|-------------------|
| **Info** | > 30% available | Normal operation | Monitor and log |
| **Warning** | 20–30% available | Swap usage increasing | Investigate within 4 hours |
| **Critical** | 10–20% available | Swap thrashing; latency spikes | Immediate intervention |
| **Emergency** | < 10% available or OOM triggered | Service outage; process terminated | Escalate within 15 minutes |

### 2.3 Root Cause Analysis

| Root Cause | Frequency | Prevention Strategy |
|------------|-----------|---------------------|
| Undersized instance (t2.micro) | High | Right-size to t3.small or larger |
| Multiple Uvicorn workers | High | Limit to `--workers 1` on small instances |
| Unbounded concurrent requests | Medium | Set `--limit-concurrency` |
| Memory leak in application | Low | Profile with `tracemalloc`; add memory guard |
| Other processes consuming RAM | Medium | Audit running services; remove unnecessary software |
| Swap disabled or too small | Medium | Add 2 GB swap; set `vm.swappiness=10` |

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
# - 502 Bad Gateway = nginx can't reach uvicorn (worker killed)
# - Connection refused = uvicorn is not running at all
# - 503 Service Unavailable = application alive but unhealthy

# Step 3: Check service status
sudo systemctl status storage-breaker
# Status indicators:
# - active (running) = service operational, issue may be external
# - inactive/failed = service down, requires immediate attention
# - "Main process exited, code=killed, status=9/KILL" = OOM killed
```

### 3.3 Phase 2: System Triage (Next 2 Minutes)

**Objective:** Identify the failure category through systematic resource assessment.

```bash
# Execute diagnostic commands
free -h            # Memory utilization
df -h /            # Disk space (rule out disk exhaustion)
uptime             # System load
pgrep -af uvicorn  # Process verification
```

**Diagnostic Interpretation:**

| Memory Status | Swap Status | Disk Status | Load Status | Likely Root Cause |
|---------------|-------------|-------------|-------------|-------------------|
| < 50 MiB free | Swap full or missing | Normal | Normal or High | **Memory exhaustion** |
| Normal | Low usage | Normal | Normal | Application defect or configuration issue |
| Normal | Normal | ≥ 95% usage | Normal | Disk exhaustion (see disk-exhaustion-report.md) |
| Normal | Normal | Normal | Very High (> 4.0) | CPU-bound process or infinite loop |

### 3.4 Phase 3: Memory Exhaustion Deep Dive

**Objective:** Identify the specific process consuming memory and confirm OOM Killer activity.

```bash
# Step 1: Check for OOM Killer activity
dmesg | grep -i "oom\|killed\|out of memory"
```

**Expected Output (OOM confirmed):**
```
[12345.678] Out of memory: Killed process 1234 (python3) total-vm:1234567kB, anon-rss:1023456kB, file-rss:0kB, shmem-rss:0kB
[12345.679] oom_reaper: reaped process 1234 (python3), now anon-rss:0kB
```

```bash
# Step 2: Check memory consumers by RSS
ps aux --sort=-%mem | head -10
```

**Expected Output:**
```
USER       PID  %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
ubuntu    1234  45.2 68.5 1234567 685432 ?      Sl   10:00  15:23 python3 -m uvicorn app:app
root        567  2.1  8.3  83456  83456 ?        Ss   09:58   1:12 /usr/sbin/nginx
```

```bash
# Step 3: Check swap usage
swapon --show
# If no output → no swap configured (high risk)
# If swap is full → system was under severe memory pressure

# Step 4: Check systemd death record
sudo journalctl -u storage-breaker --since "30 min ago" --no-pager | tail -20
```

**Expected Output:**
```
Jul 17 10:05:23 ip-10-0-1-45 systemd[1]: storage-breaker.service: Main process exited, code=killed, status=9/KILL
Jul 17 10:05:23 ip-10-0-1-45 systemd[1]: storage-breaker.service: Failed with result 'signal'.
Jul 17 10:05:28 ip-10-0-1-45 systemd[1]: storage-breaker.service: Scheduled restart job, restart counter is at 3.
```

```bash
# Step 5: Check process count
pgrep -c uvicorn
# Expected: 1 (single worker)
# If > 1: multiple workers are consuming memory — reduce to 1

# Step 6: Check the service file for worker configuration
grep -i "workers\|ExecStart" /etc/systemd/system/storage-breaker.service
```

**Root Cause Confirmed:** The system has insufficient RAM for the configured workload, and the OOM Killer terminated the Uvicorn worker process.

### 3.5 Phase 4: Cascade Damage Assessment

**Objective:** Evaluate secondary system impacts before declaring resolution.

```bash
# Verify OOM killed only the application (not sshd, nginx, etc.)
dmesg | grep -i "killed process" | awk '{print $NF}' | sort | uniq -c | sort -rn

# Check if other services were affected
sudo systemctl status nginx sshd cron

# Check for swap thrashing (high si/so values)
vmstat 1 5
# Look at si (swap in) and so (swap out) columns
# High values (> 100) indicate swap thrashing

# Verify no zombie processes left behind
ps aux | awk '{if ($8 ~ /Z/) print}'

# Check kernel logs for additional OOM events
sudo journalctl -k --since "1 hour ago" | grep -i "oom\|kill"
```

### 3.6 Phase 5: Evidence Documentation

**Objective:** Capture system state for post-incident analysis.

```bash
# Document current state
free -h > /tmp/rca-memory.log
ps aux --sort=-%mem > /tmp/rca-processes.log
dmesg | grep -i "oom\|killed" > /tmp/rca-oom.log
swapon --show > /tmp/rca-swap.log
sudo journalctl -u storage-breaker --since "30 min ago" --no-pager > /tmp/rca-journal.log
cat /etc/systemd/system/storage-breaker.service > /tmp/rca-service-file.log
```

---

## 4. Emergency Recovery Procedures

### 4.1 Objective

Restore service availability within 2 minutes when the OOM Killer has terminated the Uvicorn worker.

### 4.2 Pre-Recovery Checklist

Document the current system state before initiating recovery:

```bash
free -h > /tmp/pre-recovery-memory.log
dmesg | grep -i "oom\|killed" > /tmp/pre-recovery-oom.log
ps aux --sort=-%mem > /tmp/pre-recovery-processes.log
sudo ss -lntp > /tmp/pre-recovery-ports.log
```

### 4.3 Primary Recovery Option: Service Restart + Emergency Swap

**Rationale:** The OOM Killer terminated the process because no swap existed as a safety buffer. Adding swap prevents immediate recurrence.

**Cost:** One-time EBS cost for swap file (e.g., 2 GB = ~$0.16/month).

**Prerequisites:**
- SSH access to the instance
- Sufficient free disk space (2–4 GB for swap file)

**Procedure:**

```bash
# Step 1: Add emergency swap (2 GB)
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Step 2: Make swap persistent across reboots
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Step 3: Set low swappiness (prefer RAM, use swap as safety net)
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
sudo sysctl vm.swappiness=10

# Step 4: Verify swap is active
free -h
# Expected output:
#               total   used   free   shared  buff/cache  available
# Mem:          985Mi   350Mi   200Mi    8Mi      435Mi      500Mi
# Swap:         2.0Gi     0B   2.0Gi

# Step 5: Restart the service
sudo systemctl restart storage-breaker
sleep 3

# Step 6: Verify service health
curl -i http://EC2_PUBLIC_IP/health
# Expected: HTTP/1.1 200 OK

# Step 7: Confirm process is running
pgrep -af uvicorn
# Expected: single uvicorn process with moderate RSS
```

### 4.4 Fallback Recovery Option: Systemd Memory Limit + Restart

**Rationale:** When swap is not enough or you want deterministic failure instead of random OOM kills. Systemd kills the process cleanly, triggering an automatic restart.

**Use Case:** Recurring OOM kills despite swap; want predictable behavior.

```bash
# Step 1: Add memory limits to the service
sudo systemctl edit storage-breaker
```

Add the following:
```ini
[Service]
MemoryMax=768M
MemoryHigh=512M
```

**Memory Limit Semantics:**

| Directive | Behavior |
|-----------|----------|
| `MemoryHigh=512M` | Soft limit — system applies memory pressure above this threshold |
| `MemoryMax=768M` | Hard limit — systemd kills the process immediately above this |

```bash
# Step 2: Reload systemd
sudo systemctl daemon-reload

# Step 3: Restart the service
sudo systemctl restart storage-breaker
sleep 3

# Step 4: Verify service health
curl -i http://EC2_PUBLIC_IP/health
# Expected: HTTP/1.1 200 OK

# Step 5: Confirm memory limits are applied
systemctl show storage-breaker | grep -i "memorymax\|memoryhigh"
# Expected:
# MemoryMax=806293504
# MemoryHigh=536870912
```

### 4.5 Post-Recovery Verification

- [ ] Memory available (`free -h` shows adequate free RAM)
- [ ] Swap configured (`swapon --show` shows active swap)
- [ ] Application restarted (`systemctl status storage-breaker` shows active)
- [ ] Health endpoint returns `200 OK`
- [ ] Exactly one Uvicorn worker running
- [ ] Service stable for 5 minutes without OOM events
- [ ] No additional OOM kills in `dmesg`

---

## 5. Long-Term Prevention

### 5.1 Objective

Eliminate the possibility of memory exhaustion causing service outages by right-sizing infrastructure and adding protective resource limits.

### 5.2 Prevention Strategy

```
Instance Right-Sizing (primary defense)
    │
    ├──→ t3.small or larger (2+ GiB RAM)
    │
    └──→ Systemd Memory Guard (defense in depth)
              │
              ├──→ MemoryMax=768M (hard limit)
              ├──→ MemoryHigh=512M (soft limit, triggers pressure)
              └──→ Clean restart instead of OOM kill
```

**Key Principle:** Right-sizing eliminates the capacity gap. Systemd memory limits ensure deterministic failure when the gap reappears (e.g., traffic spike, code regression).

### 5.3 Primary Solution: Instance Right-Sizing + Systemd Memory Guard

**Rationale:** The most reliable fix — provide enough RAM so memory exhaustion is unlikely, and cap the application so it never consumes all system RAM.

**Step 1: Right-Size the Instance**

```bash
# Stop the instance
aws ec2 stop-instances --instance-ids i-XXXXXXXXXXXXXXXXX

# Modify instance type (t2.micro → t3.small)
aws ec2 modify-instance-attribute \
    --instance-id i-XXXXXXXXXXXXXXXXX \
    --instance-type t3.small

# Start the instance
aws ec2 start-instances --instance-ids i-XXXXXXXXXXXXXXXXX
```

**Memory Comparison:**

| Resource | t2.micro | t3.small |
|----------|----------|----------|
| Total RAM | 1 GiB | 2 GiB |
| OS + System Services | ~300 MiB | ~300 MiB |
| Available for Application | ~700 MiB | ~1.7 GiB |
| Safety Margin | ❌ None | ✅ ~1 GiB |
| Burst CPU | ❌ No | ✅ Yes |

**Step 2: Add Systemd Memory Guard**

```bash
sudo systemctl edit storage-breaker
```

Add:
```ini
[Service]
MemoryMax=1024M
MemoryHigh=768M
CPUQuota=80%
LimitNOFILE=4096
LimitNPROC=256
```

**Step 3: Add Uvicorn Concurrency Limit**

```bash
# Edit the ExecStart line to add concurrency limits
sudo systemctl edit storage-breaker --force
```

```ini
[Service]
ExecStart=
ExecStart=/opt/storage-breaker/venv/bin/uvicorn app:app \
    --host 127.0.0.1 \
    --port 3000 \
    --workers 1 \
    --limit-concurrency 50 \
    --timeout-keep-alive 30 \
    --backlog 20
```

**Step 4: Enable and Verify**

```bash
sudo systemctl daemon-reload
sudo systemctl restart storage-breaker
sleep 3

# Verify memory limits
systemctl show storage-breaker | grep -E "MemoryMax|MemoryHigh"
```

**Comparison: Memory Limit Approaches**

| Factor | No Limits | Swap Only | Systemd MemoryMax | Right-Sized + MemoryMax |
|--------|-----------|-----------|-------------------|------------------------|
| Prevents OOM Kill | ❌ | ⚠️ Delays | ✅ Deterministic | ✅ Deterministic |
| Service Recovery Time | 30–60s (OOM) | 30–60s (OOM) | 5–10s (systemd restart) | 5–10s (systemd restart) |
| Root Cause Visibility | ❌ Random process killed | ❌ Masked | ✅ Logged in journal | ✅ Logged in journal |
| Cost | $0 | ~$0.16/month | $0 | ~$15/month (instance) |
| Reliability | Low | Medium | High | Very High |

### 5.4 Fallback Solution: Application-Level Memory Profiling

**Rationale:** Identify and fix memory leaks at the code level when infrastructure changes are not feasible.

**Implementation:**

```python
import tracemalloc
import psutil
import os

MEMORY_THRESHOLD_MB = 512
CHECK_INTERVAL_SECONDS = 30

def get_process_memory_mb() -> float:
    """Get current RSS memory usage in MB."""
    process = psutil.Process(os.getpid())
    return process.memory_info().rss / (1024 * 1024)

def monitor_memory():
    """Log memory usage periodically for profiling."""
    tracemalloc.start()
    while True:
        current_mb = get_process_memory_mb()
        peak_mb = tracemalloc.get_traced_memory()[1] / (1024 * 1024)
        print(f"Memory: current={current_mb:.1f}MB peak={peak_mb:.1f}MB")
        if current_mb > MEMORY_THRESHOLD_MB:
            print(f"WARNING: Memory usage {current_mb:.1f}MB exceeds threshold {MEMORY_THRESHOLD_MB}MB")
        time.sleep(CHECK_INTERVAL_SECONDS)
```

---

## 6. Decision Matrix

### 6.1 Recovery Decision Tree

| Scenario | Temporary Recovery | Long-Term Prevention | Total Monthly Cost | Service Impact |
|----------|-------------------|----------------------|--------------------|----------------|
| **Single Worker, Low Traffic** | Restart + add 2GB swap | Right-size to t3.small | ~$15.34/month | ✅ Minimal |
| **High Memory Pressure** | Systemd MemoryMax + restart | Right-size to t3.medium | ~$30.69/month | ✅ Minimal |
| **Cannot Change Instance Type** | Systemd MemoryMax + swap | Memory profiling + concurrency limit | ~$0.16/month | ⚠️ Limited capacity |
| **Recurring OOM (Not Recommended)** | Systemd MemoryMax + swap | Add more swap + lower swappiness | ~$0.32/month | ⚠️ Performance degradation |

**Critical Note:** Swap is a temporary buffer, not a solution. Right-sizing is the only reliable long-term fix for memory exhaustion.

---

## 7. Monitoring & Alerting Strategy

### 7.1 Alert Thresholds

| Alert Level | Memory Usage Threshold | Response Time | Required Action |
|-------------|----------------------|---------------|-----------------|
| **Info** | > 70% used | 24 hours | Log event, no immediate action |
| **Warning** | > 80% used | 4 hours | Notify on-call, investigate |
| **Critical** | > 90% used | 1 hour | Page on-call, immediate intervention |
| **Emergency** | OOM Kill detected | 15 minutes | Auto-restart or escalation |

### 7.2 Alert Channel Configuration

| Channel | Use Case | Integration Method |
|---------|----------|-------------------|
| Email | Non-urgent warnings | AWS SNS → Email |
| Slack | Team notifications | AWS SNS → Lambda → Slack Webhook |
| PagerDuty | Critical/Emergency | AWS SNS → PagerDuty Integration |
| SMS | After-hours emergencies | AWS SNS → SMS |

### 7.3 Runbook Integration

When memory alerts trigger:
1. Open the **Health Check Runbook** (`docs/health-check-runbook.md`)
2. Execute **Phase 1: Confirm the Problem**
3. Proceed to **Phase 2: Quick Triage** — focus on `free -h` and `dmesg | grep -i oom`
4. Apply resolution based on **Decision Matrix** (Section 6)

---

## 8. Capacity Planning

### 8.1 Memory Usage Forecasting

| Metric | Warning Threshold | Critical Threshold | Required Action |
|--------|-------------------|-------------------|-----------------|
| Memory Usage | > 70% | > 85% | Plan instance upgrade or add memory guard |
| Swap Usage | > 50% | > 80% | Swap is not a solution — right-size immediately |
| OOM Kill Count | 1 per week | 3 per week | Immediate instance upgrade required |

### 8.2 Growth Rate Measurement

```bash
# Measure memory growth over 1 hour
START_RSS=$(ps -o rss= -p $(pgrep -f uvicorn) 2>/dev/null)
START_SWAP=$(free -m | awk '/Swap/{print $3}')
sleep 3600
END_RSS=$(ps -o rss= -p $(pgrep -f uvicorn) 2>/dev/null)
END_SWAP=$(free -m | awk '/Swap/{print $3}')

RSS_GROWTH=$((END_RSS - START_RSS))
SWAP_GROWTH=$((END_SWAP - START_SWAP))

echo "RSS growth: ${RSS_GROWTH} KB over 1 hour"
echo "Swap growth: ${SWAP_GROWTH} MB over 1 hour"
```

### 8.3 Capacity Recommendations

| Traffic Level | Instance Type | RAM | Swap | Systemd MemoryMax |
|---------------|--------------|-----|------|--------------------|
| Development/testing | t2.micro | 1 GiB | 2 GB | 768M |
| Low traffic (< 100 req/s) | t3.small | 2 GiB | 2 GB | 1024M |
| Medium traffic (100–500 req/s) | t3.medium | 4 GiB | Optional | 2048M |
| High traffic (> 500 req/s) | t3.large | 8 GiB | None needed | 4096M |

---

## 9. Security Considerations

| Security Concern | Mitigation Strategy |
|------------------|---------------------|
| Swap File Permissions | `chmod 600 /swapfile` — only root can read/write |
| Service Execution | Non-root user (`User=ubuntu` in systemd unit) |
| Memory Limits | Systemd `MemoryMax` prevents runaway processes |
| CPU Limits | Systemd `CPUQuota` prevents CPU starvation |
| Process Limits | `LimitNPROC` prevents fork bombs |
| File Descriptor Limits | `LimitNOFILE` prevents fd exhaustion |

---

## 10. Incident Response Process

### 10.1 Escalation Matrix

| Severity Level | Response Time | Escalation Path |
|----------------|---------------|-----------------|
| **P1 — OOM Kill (Service Down)** | 15 minutes | Immediately page on-call engineer |
| **P2 — High Memory Usage (> 90%)** | 1 hour | Notify on-call via Slack |
| **P3 — Swap Increasing** | 4 hours | Log ticket, investigate next business day |

### 10.2 Communication Template

```
Subject: [P1] Storage Breaker Health Check Failure — Memory Exhaustion / OOM Kill

Impact: Health check returning HTTP 502; OOM Killer terminated Uvicorn worker
Duration: [START_TIME] — [END_TIME] UTC
Root Cause: Memory exhaustion on t2.micro instance; OOM Killer killed process
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
- HH:MM — Memory usage exceeded threshold
- HH:MM — OOM Killer terminated uvicorn
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
| Mean Time to Recovery (MTTR) | < 5 minutes (auto-restart) | |
| Monthly OOM Kill Incidents | 0 | |
| Alert Coverage | 100% of instances | |

---

## 12. References

| Document | Description |
|----------|-------------|
| [Deployment Guide](deployment.md) | Infrastructure setup and application deployment |
| [API Reference](api-reference.md) | Application endpoint documentation |
| [Systemd Guide](systemd-guide.md) | Service management, resource limits, and security |
| [Nginx Guide](nginx-guide.md) | Reverse proxy, HTTPS, Let's Encrypt, ACM |
| [Health Check Runbook](health-check-runbook.md) | Troubleshooting, RCA, and recovery procedures |
| [Disk Exhaustion Report](disk-exhaustion-report.md) | Related report: disk exhaustion detection and recovery |

---

## Appendix A: Quick Reference Commands

| Task | Command |
|------|---------|
| Check memory usage | `free -h` |
| Check swap | `swapon --show` |
| Find memory hogs | `ps aux --sort=-%mem \| head -10` |
| Check for OOM kills | `dmesg \| grep -i "oom\|killed"` |
| Check systemd death record | `sudo journalctl -u storage-breaker -n 20` |
| Test health endpoint | `curl -i http://127.0.0.1:3000/health` |
| Check service status | `sudo systemctl status storage-breaker` |
| Add emergency swap | `sudo fallocate -l 2G /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile` |
| Check memory limits | `systemctl show storage-breaker \| grep -i memory` |
| Monitor swap activity | `vmstat 1 5` |

---

## Appendix B: Alarm Configuration Examples

### AWS CLI — CloudWatch Alarm (Memory Utilization)

```bash
# Install CloudWatch agent first, then create alarm
aws cloudwatch put-metric-alarm \
    --alarm-name "MemoryCritical-$(curl -s http://169.254.169.254/latest/meta-data/instance-id)" \
    --alarm-description "Memory utilization >= 90% for 5 minutes" \
    --metric-name MemoryUtilization \
    --namespace System/Linux \
    --statistic Average \
    --period 300 \
    --threshold 90 \
    --comparison-operator GreaterThanOrEqualToThreshold \
    --evaluation-periods 1 \
    --alarm-actions arn:aws:sns:us-east-1:ACCOUNT:memory-alerts \
    --dimensions Name=InstanceId,Value=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
```

### AWS CLI — CloudWatch Alarm (OOM Kill Detection via Custom Metric)

```bash
# Create a custom metric for OOM kills (pushed by a cron job)
aws cloudwatch put-metric-alarm \
    --alarm-name "OOMKill-$(curl -s http://169.254.169.254/latest/meta-data/instance-id)" \
    --alarm-description "OOM Kill detected on instance" \
    --metric-name OOMKillCount \
    --namespace StorageBreaker \
    --statistic Sum \
    --period 300 \
    --threshold 1 \
    --comparison-operator GreaterThanOrEqualToThreshold \
    --evaluation-periods 1 \
    --alarm-actions arn:aws:sns:us-east-1:ACCOUNT:memory-alerts \
    --dimensions Name=InstanceId,Value=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
```

### Terraform — CloudWatch Alarm

```hcl
resource "aws_cloudwatch_metric_alarm" "memory_critical" {
  alarm_name          = "MemoryCritical-${aws_instance.app.id}"
  alarm_description   = "Memory utilization >= 90% for 5 minutes"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 1
  metric_name         = "MemoryUtilization"
  namespace           = "System/Linux"
  period              = 300
  statistic           = "Average"
  threshold           = 90
  alarm_actions       = [aws_sns_topic.memory_alerts.arn]

  dimensions = {
    InstanceId = aws_instance.app.id
  }
}
```

### Systemd Timer — OOM Kill Monitor

```bash
# Create OOM monitoring script
sudo tee /usr/local/bin/oom-monitor.sh << 'EOF'
#!/bin/bash
OOM_COUNT=$(dmesg | grep -c "Out of memory")
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)

if [ "$OOM_COUNT" -gt 0 ]; then
    aws cloudwatch put-metric-data \
        --namespace StorageBreaker \
        --metric-name OOMKillCount \
        --dimensions InstanceId=$INSTANCE_ID \
        --value $OOM_COUNT \
        --unit Count
fi
EOF

sudo chmod +x /usr/local/bin/oom-monitor.sh

# Create systemd timer
sudo tee /etc/systemd/system/oom-monitor.timer << EOF
[Unit]
Description=OOM Kill Monitor

[Timer]
OnBootSec=5min
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target
EOF

sudo tee /etc/systemd/system/oom-monitor.service << EOF
[Unit]
Description=OOM Kill Monitor Service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/oom-monitor.sh
EOF

sudo systemctl enable oom-monitor.timer
sudo systemctl start oom-monitor.timer
```

---

**Document Control**

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | July 2026 | SRE Team | Initial release |

---

**END OF DOCUMENT**
