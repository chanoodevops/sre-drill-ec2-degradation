# Assessment 1: EC2 Performance Degradation Drill

## Objective

Demonstrate the ability to deploy, monitor, troubleshoot, and recover a FastAPI application experiencing disk exhaustion on an AWS EC2 instance.

## Scenario

A FastAPI application ("Storage Breaker") is deployed on an EC2 `t2.micro` instance. The application intentionally writes large log files to simulate disk exhaustion. As the root filesystem fills up, the application health check transitions from healthy to unhealthy.

Your task is to:

1. Deploy the application following the provided specifications
2. Monitor the system as disk usage increases
3. Diagnose the cause of the 503 health check failure
4. Execute recovery procedures to restore service
5. Document your findings and actions

## Requirements

### Infrastructure

| Component | Specification |
| --------- | ------------- |
| EC2 instance | `t2.micro` |
| Operating system | Ubuntu 22.04 LTS |
| Root EBS | 10 GiB `gp3` |
| Application | FastAPI + Uvicorn |
| Reverse proxy | Nginx (port 80) |
| Workers | 1 (single worker required) |

### Security Group

| Port | Source | Purpose |
| ---- | ------ | ------- |
| TCP 22 | Student IP | SSH access |
| TCP 80 | Student IP | HTTP via Nginx |

Port 3000 must not be publicly exposed.

### Application

- FastAPI application listening on `127.0.0.1:3000`
- Nginx reverse proxy on port 80
- Logs written to `/var/log/storage-breaker/application.log`
- Health endpoint at `GET /health`
- Logs generate continuously until disk fills

## Health Check Behavior

The `/health` endpoint evaluates:

1. **Disk usage** — Returns unhealthy if usage ≥ 95%
2. **Log writer state** — Returns unhealthy if status is `disk_full` or `write_failed`

| HTTP Status | Response | Meaning |
| ----------- | -------- | ------- |
| `200` | `{"status":"healthy"}` | System operating normally |
| `503` | `{"status":"unhealthy"}` | Disk exhaustion or write failure detected |

## Assessment Tasks

### Task 1: Deployment (30 points)

Deploy the application according to the [Deployment Guide](docs/deployment.md).

Verify:

- [ ] Application running on port 3000
- [ ] Nginx proxying port 80 to 3000
- [ ] Health endpoint returns `200 OK`
- [ ] Only one Uvicorn worker running
- [ ] Logs writing to `/var/log/storage-breaker`

### Task 2: Monitoring (20 points)

Monitor the system as disk usage increases.

Document:

- [ ] Initial disk usage
- [ ] Disk usage at regular intervals
- [ ] Log file growth rate
- [ ] Health check status over time
- [ ] Commands used for monitoring

### Task 3: Diagnosis (20 points)

When the health check returns 503, diagnose the root cause.

Document:

- [ ] Symptom observed (503 response)
- [ ] Diagnostic commands executed
- [ ] Root cause identified (disk exhaustion)
- [ ] Evidence collected (df output, log file size)

### Task 4: Recovery (20 points)

Execute recovery procedures to restore service.

Document:

- [ ] Recovery steps performed
- [ ] Disk space reclaimed
- [ ] Service restarted
- [ ] Health check returns 200
- [ ] Verification commands and output

### Task 5: Documentation (10 points)

Provide a clear summary of the entire drill.

Include:

- [ ] Timeline of events
- [ ] Commands used with output
- [ ] Lessons learned
- [ ] Recommendations for prevention

## Deliverables

Submit the following:

1. **Deployment verification** — Screenshots or command output showing successful deployment
2. **Monitoring logs** — Record of disk usage observations over time
3. **Diagnosis report** — Steps taken to identify the root cause
4. **Recovery report** — Steps taken to restore service
5. **Summary** — Overall assessment of the drill

## Grading Criteria

| Criteria | Points | Description |
| -------- | ------ | ----------- |
| Deployment | 30 | Correct setup following specifications |
| Monitoring | 20 | Systematic observation and documentation |
| Diagnosis | 20 | Accurate root cause identification |
| Recovery | 20 | Effective service restoration |
| Documentation | 10 | Clear, professional documentation |

**Total: 100 points**

## Tips

- Use `df -h` frequently to track disk usage
- Monitor the log file size with `watch -n 2 'ls -lh /var/log/storage-breaker/application.log'`
- Check service status with `sudo systemctl status storage-breaker`
- Verify worker count with `pgrep -af uvicorn`
- Test health with `curl -i http://EC2_PUBLIC_IP/health`

## References

- [Deployment Guide](docs/deployment.md)
- [API Reference](docs/api-reference.md)
- [Monitoring Guide](docs/monitoring.md)
- [Troubleshooting Guide](docs/troubleshooting.md)
