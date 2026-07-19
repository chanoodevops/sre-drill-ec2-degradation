# SRE Drill: EC2 Performance Degradation

A controlled SRE drill that simulates disk exhaustion on an AWS EC2 instance to practice incident response, troubleshooting, and recovery procedures.

## Overview

This application intentionally writes large log files to fill the root filesystem, triggering a health check failure. The exercise tests your ability to:

- Detect service degradation via health endpoints
- Diagnose root cause (disk exhaustion)
- Execute recovery procedures under pressure
- Implement preventive measures

## Architecture

```
                    Internet
                        │
                        ▼
                  EC2 Public IP
                        │
                     Port 80
                        │
                    +---------+
                    | Nginx   |  ← Reverse proxy
                    +---------+
                        │
                http://127.0.0.1:3000
                        │
                  +-------------+
                  | Uvicorn     |  ← Workers = 1
                  | (FastAPI)   |
                  +-------------+
                        │
              /var/log/storage-breaker
                        │
                application.log  ← Grows to ~10 GiB
```

## AWS Specifications

| Resource         | Specification           |
| ---------------- | ----------------------- |
| EC2 instance     | `t2.micro`              |
| Operating system | Ubuntu 22.04 LTS (recommended) or 24.04 LTS |
| Root EBS volume  | 10 GiB (`gp3`)         |
| Public IPv4      | Enabled                 |
| Uvicorn workers  | 1 (required)            |

### Security Group

| Port | Source | Purpose |
| ---- | ------ | ------- |
| TCP 22 | Your IP | SSH |
| TCP 80 | Your IP | Nginx HTTP |

> **Note:** Port 3000 must not be exposed publicly. Uvicorn listens only on `127.0.0.1`.

## Quick Start

### 1. Connect to EC2

```bash
chmod 400 <your-key>.pem
ssh-add <your-key>.pem
ssh ubuntu@EC2_PUBLIC_IP
```

### 2. Install Dependencies

```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-pip git nginx
```

### 3. Deploy Application

```bash
git clone <repository-url> /opt/storage-breaker
cd /opt/storage-breaker

python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 4. Create Log Directory

```bash
sudo mkdir -p /var/log/storage-breaker
sudo chown -R ubuntu:ubuntu /var/log/storage-breaker
sudo chmod 750 /var/log/storage-breaker
```

### 5. Start Application

```bash
.venv/bin/uvicorn app:app \
  --host 127.0.0.1 \
  --port 3000 \
  --workers 1 \
  --no-access-log
```

### 6. Configure Nginx

Configure Nginx as a reverse proxy from port 80 to `http://127.0.0.1:3000`. See [Deployment Guide](docs/deployment.md) for full configuration.

### 7. Verify

```bash
curl -i http://EC2_PUBLIC_IP/health
```

Expected response:

```
HTTP/1.1 200 OK
{"status":"healthy"}
```

## Health Check Behavior

The `/health` endpoint returns:

| Status Code | Response | Condition |
| ----------- | -------- | --------- |
| `200` | `{"status":"healthy"}` | Disk usage < 95%, no write errors |
| `503` | `{"status":"unhealthy"}` | Disk usage ≥ 95% or write errors |

## Documentation

| Document | Description |
| -------- | ----------- |
| [Deployment Guide](docs/deployment.md) | Infrastructure setup and application deployment |
| [API Reference](docs/api-reference.md) | Application endpoint documentation |
| [Systemd Guide](docs/systemd-guide.md) | Service management, logging, and security |
| [Nginx Guide](docs/nginx-guide.md) | Reverse proxy, HTTPS, Let's Encrypt, ACM |
| [Health Check Runbook](docs/health-check-runbook.md) | Troubleshooting, RCA, and recovery procedures |
| [Disk Exhaustion Report](docs/disk-exhaustion-report.md) | Enterprise guide: detection, recovery, and prevention |

## Important Constraints

- **Single Worker:** Do not use more than one Uvicorn worker. Each worker starts an additional log-generation thread.
- **Port Isolation:** Uvicorn must listen on `127.0.0.1` only. Nginx handles public traffic.
- **Log Location:** Application logs write to `/var/log/storage-breaker/application.log`.

## License

Internal training exercise.
