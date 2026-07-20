# Health Check Failure Scenarios — Reference

## Overview

This document lists all known health check failure scenarios for the Storage Breaker application. Each scenario can be developed into a full enterprise incident report and runbook section.

**Health check behavior:** `GET /health` returns `200 OK` when healthy, `503 Service Unavailable` when unhealthy (disk ≥ 95%, write errors, or process terminated).

---

## Scenario Inventory

| # | Scenario | Symptom | Root Cause | Diagnostic Command |
|---|----------|---------|------------|-------------------|
| 1 | **Disk Exhaustion** | 503, writer status `disk_full` | Log file fills root volume | `df -h /` ≥ 95% |
| 2 | **Memory Exhaustion / OOM Kill** | 502/503, connection refused | OOM Killer terminated Uvicorn worker | `dmesg \| grep -i oom` |
| 3 | **File Descriptor Exhaustion** | 503, `Too many open files` in logs | fd leak or high concurrency exceeds ulimit | `cat /proc/sys/fs/file-nr` |
| 4 | **Nginx Misconfiguration** | 502 Bad Gateway | Bad upstream block, proxy_pass error | `nginx -t`, `curl localhost:3000/health` |
| 5 | **SSL/TLS Certificate Expiration** | HTTPS health checks fail, browser warnings | Let's Encrypt cert expired, auto-renewal broken | `openssl s_client -connect :443` |
| 6 | **Port Conflict** | Service won't start, 502 from nginx | Another process bound to port 3000 | `ss -lntp \| grep :3000` |
| 7 | **Network ACL / Security Group** | Timeout from external, works on localhost | Firewall blocks ALB health check probes | `telnet` from external, check SG/NACL |
| 8 | **ALB Target Group Misconfiguration** | ALB shows unhealthy, app is fine | Wrong health check path, port, or protocol | `aws elbv2 describe-target-health` |
| 9 | **Swap Thrashing** | 503, extreme slowness, high si/so | No RAM, excessive swap I/O, CPU stalled | `vmstat 1 5`, `free -h` |
| 10 | **Application Dependency Failure** | 503, app returns unhealthy | External service (DB, cache) unreachable | App logs, dependency health endpoint |
| 11 | **Clock Skew** | Intermittent failures, SSL errors | NTP not synced, time drifts minutes/hours | `timedatectl status`, `ntpstat` |
| 12 | **Zombie Process Accumulation** | Slow degradation, eventually 503 | Zombie processes consuming PID slots | `ps aux \| awk '$8 ~ /Z/'` |

---

## Reports and Runbook Sections

| Scenario | Report | Runbook Step | Status |
|----------|--------|-------------|--------|
| Disk Exhaustion | [disk-exhaustion-report.md](disk-exhaustion-report.md) | Step A | ✅ Complete |
| Memory Exhaustion | [memory-exhaustion-report.md](memory-exhaustion-report.md) | Step B | ✅ Complete |
| File Descriptor Exhaustion | — | — | 🔲 Pending |
| Nginx Misconfiguration | — | Step F (partial) | 🔲 Pending |
| SSL/TLS Certificate Expiration | — | — | 🔲 Pending |
| Port Conflict | — | Step D (partial) | 🔲 Pending |
| Network ACL / Security Group | — | — | 🔲 Pending |
| ALB Target Group Misconfiguration | — | Step G (partial) | 🔲 Pending |
| Swap Thrashing | — | — | 🔲 Pending |
| Application Dependency Failure | — | — | 🔲 Pending |
| Clock Skew | — | — | 🔲 Pending |
| Zombie Process Accumulation | — | — | 🔲 Pending |

---

## Priority for Next Report

| Priority | Scenario | Rationale |
|----------|----------|-----------|
| 1 | File Descriptor Exhaustion | High frequency in production, clear diagnostics, distinct from disk/memory |
| 2 | Network ACL / Security Group | Teaches network-layer debugging, common after team changes |
| 3 | Clock Skew | Sneaky, hard to diagnose, excellent learning exercise |
| 4 | SSL/TLS Certificate Expiration | Very common, real-world impact, auto-renewal is often broken |
| 5 | Swap Thrashing | Related to memory exhaustion but different symptoms and recovery |

---

## Cross-Reference

| Related Document | Description |
|-----------------|-------------|
| [Health Check Runbook](health-check-runbook.md) | Primary troubleshooting guide with Step A–G |
| [Disk Exhaustion Report](disk-exhaustion-report.md) | Enterprise report: disk exhaustion |
| [Memory Exhaustion Report](memory-exhaustion-report.md) | Enterprise report: OOM / memory exhaustion |
| [API Reference](api-reference.md) | Health endpoint behavior and response format |
| [Deployment Guide](deployment.md) | Instance specs, nginx, systemd setup |
