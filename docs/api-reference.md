# API Reference

## Base URL

```
http://EC2_PUBLIC_IP
```

All requests pass through Nginx (port 80) which proxies to Uvicorn (port 3000 on localhost).

---

## Endpoints

### Health Check

```
GET /health
```

Returns the current health status of the application based on disk usage and log writer state.

#### Response

**Healthy (200 OK)**

```json
{
  "status": "healthy"
}
```

**Unhealthy (503 Service Unavailable)**

```json
{
  "status": "unhealthy"
}
```

#### Health Check Logic

The endpoint evaluates two conditions:

| Condition | Threshold | Result |
| --------- | --------- | ------ |
| Disk usage | ≥ 95% | Unhealthy |
| Log writer status | `disk_full` or `write_failed` | Unhealthy |

Both conditions must be clear for the service to report healthy.

#### Example Requests

```bash
# Basic health check
curl -i http://EC2_PUBLIC_IP/health

# Verbose output
curl -v http://EC2_PUBLIC_IP/health

# JSON formatting
curl -s http://EC2_PUBLIC_IP/health | python3 -m json.tool
```

#### Example Responses

```http
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)
Date: Fri, 17 Jul 2026 14:24:39 GMT
Content-Type: application/json
Content-Length: 20
Connection: keep-alive

{"status":"healthy"}
```

```http
HTTP/1.1 503 Service Unavailable
Server: nginx/1.24.0 (Ubuntu)
Date: Fri, 17 Jul 2026 14:12:59 GMT
Content-Type: application/json
Content-Length: 22
Connection: keep-alive

{"status":"unhealthy"}
```

---

## Application Behavior

### Log Generation

On startup, the application begins writing log records to:

```
/var/log/storage-breaker/application.log
```

| Parameter | Value |
| --------- | ----- |
| Record size | 64 KiB |
| Target total | ~10 GiB |
| Duration | ~2.5 hours |
| Write interval | Calculated to fill target in duration |

### Log Record Format

Each record is a single-line JSON object:

```json
{
  "timestamp": "2026-07-17T14:00:00+00:00",
  "level": "INFO",
  "service": "storage-breaker",
  "sequence": 1,
  "event": "background_processing",
  "status": "completed",
  "message": "Background processing completed successfully",
  "payload": "xxx...xxx"
}
```

### State Transitions

| State | Description |
| ----- | ----------- |
| `starting` | Application initializing |
| `writing` | Actively generating logs |
| `disk_full` | ENOSPC error encountered |
| `write_failed` | Other write error |
| `stopped` | Service shut down |

---

## Error Handling

### Disk Full (ENOSPC)

When the filesystem runs out of space:

1. Log writer enters `disk_full` state
2. Health endpoint returns `503`
3. Writer thread retries every 5 seconds
4. Recovery occurs automatically after space is freed

### Other Write Errors

For non-ENOSPC errors:

1. Log writer enters `write_failed` state
2. Health endpoint returns `503`
3. Writer thread retries after 5-second delay

---

## Security Notes

- Uvicorn listens on `127.0.0.1` only (not publicly accessible)
- Nginx serves as the public-facing reverse proxy
- Port 3000 must not be exposed in Security Groups
- No authentication is implemented (training exercise)

---

## See Also

| Document | Description |
|----------|-------------|
| [Deployment Guide](deployment.md) | Infrastructure setup and application deployment |
| [Systemd Guide](systemd-guide.md) | Service management, logging, and security |
| [Nginx Guide](nginx-guide.md) | Reverse proxy, HTTPS, Let's Encrypt, ACM |
| [Health Check Runbook](health-check-runbook.md) | Troubleshooting, RCA, and recovery procedures |
| [Disk Exhaustion Report](disk-exhaustion-report.md) | Enterprise guide: detection, recovery, and prevention |
