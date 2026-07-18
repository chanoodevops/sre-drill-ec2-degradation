# Monitoring Guide

Procedures for monitoring the Storage Breaker application and underlying infrastructure.

## Service Status

### Check systemd Service

```bash
sudo systemctl status storage-breaker
```

Expected output when running:

```
● storage-breaker.service - Storage Breaker FastAPI Service
     Loaded: loaded (/etc/systemd/system/storage-breaker.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2026-07-17 14:00:00 UTC; 5min ago
```

### View Service Logs

```bash
sudo journalctl -u storage-breaker -f
```

View last 100 lines:

```bash
sudo journalctl -u storage-breaker -n 100
```

---

## Application Logs

### Real-Time Log Monitoring

```bash
tail -f /var/log/storage-breaker/application.log
```

### Log File Size

```bash
ls -lh /var/log/storage-breaker/application.log
```

### Continuous Size Monitoring

```bash
watch -n 2 'ls -lh /var/log/storage-breaker/application.log'
```

---

## Nginx Logs

### Access Log

```bash
sudo tail -f /var/log/nginx/storage-breaker-access.log
```

### Error Log

```bash
sudo tail -f /var/log/nginx/storage-breaker-error.log
```

---

## Disk Usage

### Check Root Filesystem

```bash
df -h /
```

### Check Application Log Filesystem

```bash
df -h /var/log/storage-breaker
```

### Continuous Monitoring

```bash
watch -n 5 'df -h /var/log/storage-breaker'
```

### Check All Filesystems

```bash
df -h
```

### Inode Usage

```bash
df -ih
```

---

## Process Verification

### Verify Uvicorn Process

```bash
pgrep -af uvicorn
```

Or:

```bash
ps -ef | grep uvicorn
```

Confirm exactly **one** worker process is running.

### Check Listening Ports

```bash
sudo ss -lntp | grep -E ':80|:3000'
```

Expected output:

```
LISTEN   0   511   0.0.0.0:80    0.0.0.0:*   users:(("nginx",pid=1234,fd=6))
LISTEN   0   511   127.0.0.1:3000  0.0.0.0:*   users:(("uvicorn",pid=5678,fd=5))
```

---

## Health Check Monitoring

### Manual Check

```bash
curl -s http://127.0.0.1:3000/health | python3 -m json.tool
```

### Continuous Monitoring

```bash
watch -n 5 'curl -s http://127.0.0.1:3000/health'
```

### Check HTTP Status Code

```bash
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:3000/health
```

Returns `200` for healthy, `503` for unhealthy.

---

## Find Large Files

### Largest Files Under /var/log

```bash
sudo du -ah /var/log | sort -rh | head -20
```

### Largest Directories on Root

```bash
sudo du -xhd1 / 2>/dev/null | sort -h
```

### Check Open Deleted Files

```bash
sudo lsof +L1
```

---

## Monitoring Summary

| What to Monitor | Command | Frequency |
| --------------- | ------- |-----------|
| Service status | `systemctl status storage-breaker` | On demand |
| Service logs | `journalctl -u storage-breaker -f` | During incidents |
| Disk usage | `df -h /` | Every 5 minutes |
| Log file size | `ls -lh /var/log/storage-breaker/application.log` | Every 2 minutes |
| Process count | `pgrep -af uvicorn` | On demand |
| Health endpoint | `curl http://127.0.0.1:3000/health` | Every 5 minutes |
| Port bindings | `ss -lntp \| grep -E ':80\|:3000'` | On demand |
