# shoreline solutions — Security Services Setup Guide

## Prerequisites

Before starting this guide, complete the [Infrastructure Setup Guide](https://github.com/localguy009/aws-grc-lab-series/blob/main/Labs/Lab01-infrastructure/aws-infrastructure-setup.md). You should have the following running:

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

### Add the bucket policy

AWS services like Config write to S3 as service principals — they don't use your IAM credentials. This policy grants CloudTrail and Config permission to deliver logs to this bucket. Without it, AWS Config will fail to create its configuration recorder.

1. Open the bucket → **Permissions → Bucket Policy → Edit**
2. Paste the policy below, replacing `YOUR_ACCOUNT_ID` with your 12-digit AWS account ID and `YOUR_SUFFIX` with the suffix you chose for the bucket name
3. Click **Save changes**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSConfigBucketPermissionsCheck",
      "Effect": "Allow",
      "Principal": {
        "Service": "config.amazonaws.com"
      },
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::shoreline-audit-logs-YOUR_SUFFIX",
      "Condition": {
        "StringEquals": {
          "AWS:SourceAccount": "YOUR_ACCOUNT_ID"
        }
      }
    },
    {
      "Sid": "AWSConfigBucketDelivery",
      "Effect": "Allow",
      "Principal": {
        "Service": "config.amazonaws.com"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::shoreline-audit-logs-YOUR_SUFFIX/AWSLogs/YOUR_ACCOUNT_ID/Config/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control",
          "AWS:SourceAccount": "YOUR_ACCOUNT_ID"
        }
      }
    }
  ]
}
```

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

> **Before proceeding:** Confirm the bucket policy from Step 1 is saved. Config will fail if the policy is missing.

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

### Enable NIST 800-53 Conformance Pack

3. In the left sidebar, go to **Conformance packs**
4. Click **Deploy conformance pack**
5. Under **Template**, select **Use existing template from AWS sample templates**
6. From the dropdown, select **Operational Best Practices for NIST 800-53 rev 4**
7. Click **Next**

| Setting | Value |
|---------|-------|
| Conformance pack name | `shoreline-nist-800-53-config-rules` |
| Delivery S3 bucket | `shoreline-audit-logs` |

8. Click **Next → Deploy conformance pack**

Done. AWS Config will evaluate your resources against the NIST 800-53 Rev 4 rules. Results will appear in the Conformance packs dashboard within a few minutes.

---

## Step 5 — Enable Security Hub

Security Hub aggregates findings from GuardDuty and AWS Config into a single compliance dashboard. Enabling the NIST 800-53 standard maps your environment directly to control requirements.

> **Before proceeding:** Confirm your AWS console is set to **us-east-1** (top-right region selector). This lab only covers us-east-1 — do not enable Security Hub in other regions.

1. Go to **Security Hub → Enable Security Hub**
2. Leave the capabilities set to **Enable all capabilities** (default)
3. Under **Regions**, select **Enable specific Regions** — do NOT use "Enable in all Regions" (the recommended default)
4. Confirm that only **us-east-1** is selected
5. Click **Enable Security Hub**
6. In the left sidebar go to **CSPM → Security standards**
7. Find **NIST Special Publication 800-53 Revision 5** and click **Enable**

Done. Security Hub will begin populating findings within 30 minutes. AWS Config must be enabled and recording first for compliance checks to evaluate correctly.

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

