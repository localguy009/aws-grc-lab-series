# shoreline solutions — Security Services Setup Guide

## Prerequisites

Before starting this guide, complete the [Infrastructure Setup Guide](shoreline-infrastructure-setup.md). You should have the following running:

- VPC with public and private subnets
- Web server and app server EC2 instances
- Application Load Balancer
- RDS MySQL database

---

**Region:** us-east-1
**Account:** shoreline solutions AWS account

---

## What You Are Building

```
shoreline-audit-logs (S3)
    |
    ├── CloudTrail      — API activity logs
    ├── AWS Config      — resource configuration snapshots
    └── GuardDuty       — threat findings

Security Hub
    └── NIST SP 800-53 Rev 5 compliance dashboard
        └── (pulls from Config + GuardDuty)
```

---

## Step 1 — Create the Audit Logs S3 Bucket

This bucket receives CloudTrail logs, AWS Config snapshots, and GuardDuty findings. All security services write here.

1. Go to **S3 → Create bucket**

| Setting | Value |
|---------|-------|
| Bucket name | `shoreline-audit-logs` + a unique suffix (e.g. your account ID) |
| Region | us-east-1 |
| Object ownership | ACLs disabled |
| Block all public access | **Enabled** |

2. Click **Create bucket**

### Enable versioning

1. Open the bucket → **Properties → Bucket Versioning → Edit**
2. Select **Enable** → Save

### Enable default encryption

1. **Properties → Default encryption → Edit**
2. Encryption type: **SSE-S3** → Save

**Tags:**

| Key | Value |
|-----|-------|
| `environment` | `production` |
| `company` | `shoreline-systems` |

---

## Step 2 — Enable CloudTrail

CloudTrail records every API call made in your AWS account — who did what, when, and from where. This is the primary audit log for the account and is required by several NIST 800-53 controls. Security Hub checks for it directly.

1. Go to **CloudTrail → Create trail**

| Setting | Value |
|---------|-------|
| Trail name | `shoreline-cloudtrail` |
| Storage location | Existing S3 bucket |
| S3 bucket | `shoreline-audit-logs` |
| Log file SSE-KMS encryption | Disabled (SSE-S3 on the bucket handles this) |
| CloudWatch Logs | Disabled for now |
| Management events | Read and Write |
| Data events | None |
| Insights events | None |

2. Click **Create trail**

Done. CloudTrail is now delivering all API activity logs to your audit bucket under a `/CloudTrail/` prefix.

---

## Step 3 — Enable GuardDuty

GuardDuty is AWS's threat detection service. It analyzes VPC Flow Logs, CloudTrail events, and DNS logs automatically — no configuration required after enabling.

1. Go to **GuardDuty**
2. Click **Get Started → Enable GuardDuty**

Done. GuardDuty is now active and will surface findings in Security Hub automatically.

---

## Step 4 — Enable AWS Config

AWS Config records the configuration state of your resources over time. Security Hub uses Config rules to evaluate compliance — without it, many NIST 800-53 checks will show no data.

1. Go to **AWS Config → Get started**

| Setting | Value |
|---------|-------|
| Recording strategy | All resources |
| Include global resources | Yes |
| AWS Config role | Create a new role |
| Delivery method | S3 bucket |
| S3 bucket | `shoreline-audit-logs` (existing bucket) |
| SNS notification | Skip for now |

2. Click **Next → Next → Confirm**

Done. Config will begin recording resource configurations immediately and delivering snapshots to your audit logs bucket.

---

## Step 5 — Enable Security Hub

Security Hub aggregates findings from GuardDuty and AWS Config into a single compliance dashboard. Enabling the NIST 800-53 standard maps your environment directly to control requirements.

1. Go to **Security Hub**
2. Click **Go to Security Hub → Enable Security Hub**
3. Enable the **NIST SP 800-53 Rev 5** standard

Done. Security Hub will begin populating findings within 30 minutes. AWS Config must be enabled first for compliance checks to evaluate correctly.

---

## Security Layer Complete

All security services are running. Here is what you have:

| Service | Purpose | Status |
|---------|---------|--------|
| shoreline-audit-logs (S3) | Central log storage | Ready |
| CloudTrail | API activity audit log | Enabled |
| GuardDuty | Threat detection | Enabled |
| AWS Config | Resource configuration tracking | Enabled |
| Security Hub (NIST 800-53 Rev 5) | Compliance dashboard | Enabled |

**Next step:** Add VPC Flow Logs and the Compliance Hub Lambda to start building the automation layer on top of this environment.
