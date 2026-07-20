# CLAUDE.md

## Project Context

This repository contains **RCA (Root Cause Analysis) reports and incident response documentation** for health check failures in platform engineering. It is an SRE drill project that simulates infrastructure degradation on AWS EC2 instances to practice incident response, troubleshooting, and recovery.

**Primary scenario:** Storage Breaker application — a FastAPI service that intentionally fills disk to trigger health check failures (HTTP 503).

---

## Role

You are assisting a **Senior Platform Engineer** who also serves as a **Technical Writer**. All output must reflect enterprise-grade operational standards suitable for production SRE teams.

---

## Documentation Standards

### Report Format

All incident reports must follow the enterprise technical report format defined in `docs/disk-exhaustion-report.md` and `docs/memory-exhaustion-report.md`. Every report includes:

| Section | Required |
|---------|----------|
| Document metadata table | Yes |
| 1. Executive Summary (Purpose, Business Impact, Recommended Solutions, Key Findings, Cost Analysis) | Yes |
| 2. Background (Definition, Impact Classification, Root Cause Analysis) | Yes |
| 3. Detection Procedures (Phase 1–5 with exact commands and expected output) | Yes |
| 4. Emergency Recovery Procedures (Pre-recovery checklist, Primary option, Fallback option, Post-recovery verification) | Yes |
| 5. Long-Term Prevention (Primary solution, Fallback solution, comparison table) | Yes |
| 6. Decision Matrix | Yes |
| 7. Monitoring and Alerting Strategy (Alert thresholds, Channel configuration, Runbook integration) | Yes |
| 8. Capacity Planning (Forecasting, Growth rate measurement, Recommendations) | Yes |
| 9. Security Considerations | Yes |
| 10. Incident Response Process (Escalation matrix, Communication template) | Yes |
| 11. Post-Incident Review (Blameless post-mortem template, Key performance metrics) | Yes |
| 12. References | Yes |
| Appendix A: Quick Reference Commands | Yes |
| Appendix B: Alarm Configuration Examples (AWS CLI + Terraform) | Yes |
| Document Control table | Yes |

### Runbook Format

All runbooks must follow the format defined in `docs/health-check-runbook.md`:

- Failure Summary table (Incident, Root Causes, Impact, Severity, Affected Component)
- Quick Reference section (Troubleshooting commands table, 503 Causes table)
- Phased troubleshooting process (Phase 1–5)
- Failure Category Classification table
- Step-by-step RCA sections (Step A, Step B, etc.) for each failure type
- Temporary Resolution with multiple options (A, B, C...)
- Long-Term Resolution with numbered fixes
- Decision Matrix
- Enterprise Prevention sections
- Professional SRE Considerations (10 sections)
- Recovery Verification Checklist
- See Also references

### Writing Rules

1. **Commands with output:** Every diagnostic command must include the expected output or example output in fenced code blocks
2. **Tables over prose:** Use tables for comparisons, thresholds, and decision logic — not paragraphs
3. **Cost analysis:** Every solution must include a cost breakdown (AWS pricing where applicable)
4. **Security:** Every procedure must note permission requirements and security implications
5. **Verification:** Every procedure must end with verification steps proving the fix worked
6. **Cross-references:** Link related reports and runbooks using relative markdown links
7. **No abbreviations without definition:** Write "Root Cause Analysis (RCA)" on first use, then use "RCA"

### Markdown Formatting

- Use `---` horizontal rules between major sections
- Use `#` for report title, `##` for sections, `###` for subsections
- Use fenced code blocks with language identifiers (`bash`, `json`, `yaml`, `hcl`, `ini`, `python`)
- Use blockquotes `>` for important notes and warnings
- Use `- [ ]` for verification checklists
- Use `✅` and `❌` in tables for pass/fail indicators
- Use `**bold**` for key terms on first occurrence in each section

---

## Git Rules

### Commit Message Format

```
<type>: <subject>

<body>

Co-Authored-By: Claude <noreply@anthropic.com>
```

**Types:**

| Type | When to Use |
|------|-------------|
| `feat` | New report, new runbook section, new scenario |
| `fix` | Correcting errors in existing documentation |
| `docs` | Minor documentation updates (typos, formatting, links) |
| `refactor` | Restructuring existing documentation without content changes |
| `chore` | Metadata updates (README, CLAUDE.md, .gitignore) |

**Rules:**
- Subject line: imperative mood, lowercase, no period, max 72 characters
- Body: explain what and why, not how (the diff shows how)
- Reference related issues or reports when applicable
- Always include `Co-Authored-By: Claude <noreply@anthropic.com>`

**Examples:**

```
feat: add DNS resolution failure incident report

New enterprise report covering DNS resolution failures causing 503
health check errors on EC2 instances behind ALB.

Co-Authored-By: Claude <noreply@anthropic.com>
```

```
fix: correct OOM kill threshold in memory exhaustion report

The memory exhaustion report incorrectly stated 90% as the OOM
trigger threshold. The actual trigger is based on RSS vs available
RAM, not a percentage.

Co-Authored-By: Claude <noreply@anthropic.com>
```

---

## File Structure Rules

```
docs/
├── <topic>-report.md          # Enterprise incident reports
├── <topic>-runbook.md         # Troubleshooting runbooks (if separate)
├── health-check-runbook.md    # Primary runbook (all failure types)
├── deployment.md              # Deployment procedures
├── api-reference.md           # API documentation
├── systemd-guide.md           # Service management guide
└── nginx-guide.md             # Reverse proxy guide
```

**Naming conventions:**
- Reports: `<descriptive-name>-report.md` (kebab-case)
- Runbooks: `<descriptive-name>-runbook.md` (kebab-case)
- Guides: `<topic>-guide.md` (kebab-case)

---

## Content Rules

1. **Every report must be self-contained** — a reader should not need to open another file to understand the report
2. **Every command must be copy-pasteable** — no placeholders like `<YOUR_BUCKET>` without showing the format
3. **Every decision must have a trade-off table** — compare approaches with cost, complexity, and data safety columns
4. **Every recovery procedure must have a checklist** — verifiable items that prove the system is healthy
5. **Every long-term fix must include AWS CLI and Terraform examples** — in Appendix B
6. **Pricing must reference current AWS pricing** — S3, EC2, CloudWatch. Note the pricing date.
7. **Never assume the reader knows the tool** — explain what `df -h` shows, what each flag means

---

## Existing Reports

| Report | Failure Type | Key Diagnostic |
|--------|-------------|----------------|
| [Disk Exhaustion](docs/disk-exhaustion-report.md) | Disk full → 503 | `df -h /` shows ≥ 95% |
| [Memory Exhaustion](docs/memory-exhaustion-report.md) | OOM kill → 502/503 | `dmesg \| grep -i oom` |

When creating a new report, use the existing reports as templates and maintain consistent section numbering, table formats, and command style.
