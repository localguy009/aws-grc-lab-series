# shoreline solutions — Infrastructure Setup Guide

## Prerequisites

Before starting this guide, make sure you have the following in place:

- An AWS account created and accessible
- IAM user or role with administrative permissions (or at minimum: EC2, VPC, RDS, S3, IAM, Lambda, GuardDuty, Security Hub full access)
- AWS Management Console access via browser


---

**Region:** us-east-1
**Account:** shoreline solutions AWS account

---

## What You Are Building

```
Internet
    |
ALB (public subnets — AZ1 + AZ2)
    |
NAT Gateway (public subnet — AZ1)
    |
Web Server EC2 (private subnet — AZ1)
    |
App Server EC2 (private subnet — AZ2)
    |
RDS MySQL 8.0 (private subnet — AZ1)
```

---

## Step 1 — Create the VPC

1. Go to **VPC → Create VPC**
2. Select **VPC and more**

| Setting | Value |
|---------|-------|
| Name tag | `shoreline solutions` |
| IPv4 CIDR | `10.0.0.0/16` |
| IPv6 | None |
| Tenancy | Default |
| Availability Zones | 2 |
| Public subnets | 2 |
| Private subnets | 2 |
| NAT gateways | None |
| VPC endpoints | None |

3. Click **Create VPC**

**What gets created automatically:**

| Resource | CIDR |
|----------|------|
| VPC | 10.0.0.0/16 |
| Public Subnet AZ1 | 10.0.0.0/20 |
| Public Subnet AZ2 | 10.0.16.0/20 |
| Private Subnet AZ1 | 10.0.128.0/20 |
| Private Subnet AZ2 | 10.0.144.0/20 |
| Internet Gateway | — |
| Route Tables | — |

---

## Step 2 — Create the ALB Security Group

1. Go to **EC2 → Security Groups → Create security group**

| Setting | Value |
|---------|-------|
| Name | `shoreline-alb-sg-01` |
| Description | `Inbound HTTP from internet to Shoreline ALB` |
| VPC | `shoreline solutions-vpc` |

**Inbound rule:**

| Type | Port | Source |
|------|------|--------|
| HTTP | 80 | `0.0.0.0/0` |

2. Click **Create security group**

---

## Step 3 — Create the Web Server Security Group

1. Go to **EC2 → Security Groups → Create security group**

| Setting | Value |
|---------|-------|
| Name | `shoreline-web-ec2-sg` |
| Description | `Inbound HTTP from ALB only` |
| VPC | `shoreline solutions-vpc` |

**Inbound rule:**

| Type | Port | Source |
|------|------|--------|
| HTTP | 80 | `shoreline-alb-sg-01` (security group ID) |

2. Click **Create security group**

> Set the HTTP source to the security group ID of `shoreline-alb-sg-01` — not an IP range.

---

## Step 4 — Create the App Server Security Group

1. Go to **EC2 → Security Groups → Create security group**

| Setting | Value |
|---------|-------|
| Name | `shoreline-app-ec2-sg` |
| Description | `Inbound HTTP from web tier only` |
| VPC | `shoreline solutions-vpc` |

**Inbound rule:**

| Type | Port | Source |
|------|------|--------|
| HTTP | 80 | `shoreline-web-ec2-sg` (security group ID) |

2. Click **Create security group**

---

## Step 5 — Create the RDS Security Group

1. Go to **EC2 → Security Groups → Create security group**

| Setting | Value |
|---------|-------|
| Name | `shoreline-rds-sg` |
| Description | `Inbound MySQL from app tier only` |
| VPC | `shoreline solutions-vpc` |

**Inbound rule:**

| Type | Port | Source |
|------|------|--------|
| MySQL/Aurora | 3306 | `shoreline-app-ec2-sg` (security group ID) |

2. Click **Create security group**

---

## Step 6 — Create a NAT Gateway

The web and app servers are in private subnets with no direct internet route. A NAT Gateway in the public subnet gives them outbound internet access. User Data runs on first boot and needs this to pull packages.

**Allocate an Elastic IP:**

1. Go to **EC2 → Elastic IPs → Allocate Elastic IP address**
2. Leave defaults → click **Allocate**
3. Note the allocation ID

**Create the NAT Gateway:**

1. Go to **VPC → NAT Gateways → Create NAT Gateway**

| Setting | Value |
|---------|-------|
| Name | `shoreline-nat-gw` |
| Subnet | `shoreline solutions-subnet-public1-us-east-1a` |
| Connectivity type | Public |
| Elastic IP | Select the one you just allocated |

2. Click **Create NAT Gateway**

Wait 1–2 minutes for status to show **Available**.

> **Cost note:** NAT Gateway runs ~$1/day. Delete it when done recording.

**Update the private route table:**

1. Go to **VPC → Route Tables**
2. Select the route table associated with your private subnets (check the **Subnet Associations** tab)
3. Click **Edit routes → Add route**

| Destination | Target |
|-------------|--------|
| `0.0.0.0/0` | `shoreline-nat-gw` |

4. Click **Save changes**

---

## Step 7 — Launch the Web Server EC2

Apache is installed automatically via User Data at launch — no need to connect to the instance manually.

1. Go to **EC2 → Instances → Launch instances**

| Setting | Value |
|---------|-------|
| Name | `shoreline-web-server-01` |
| AMI | Amazon Linux 2023 |
| Instance type | `t2.micro` |
| Key pair | Create new — `shoreline-web-kp` (RSA, .pem) — download and save it |
| VPC | `shoreline solutions-vpc` |
| Subnet | `shoreline solutions-subnet-private1-us-east-1a` |
| Auto-assign public IP | **Disabled** |
| Security group | `shoreline-web-ec2-sg` |
| Storage | 8 GiB gp3, encryption enabled |

**Tags:**

| Key | Value |
|-----|-------|
| `environment` | `production` |
| `tier` | `web` |
| `company` | `shoreline-systems` |

**User Data** (under Advanced details → User data):

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>shoreline solutions - Client Portal</h1>" > /var/www/html/index.html
```

2. Click **Launch instance**

---

## Step 8 — Create the Target Group

1. Go to **EC2 → Target Groups → Create target group**

| Setting | Value |
|---------|-------|
| Target type | Instances |
| Name | `shoreline-web-tg` |
| Protocol | HTTP |
| Port | 80 |
| VPC | `shoreline solutions-vpc` |

2. Register `shoreline-web-server-01` as a target on port 80
3. Click **Create target group**

---

## Step 9 — Create the Application Load Balancer

1. Go to **EC2 → Load Balancers → Create load balancer**
2. Select **Application Load Balancer**

| Setting | Value |
|---------|-------|
| Name | `shoreline-alb-prod-web-east` |
| Scheme | Internet-facing |
| IP address type | IPv4 |
| VPC | `shoreline solutions-vpc` |
| Subnets | Both public subnets |
| Security group | `shoreline-alb-sg-01` (remove default) |
| Listener | HTTP :80 → forward to `shoreline-web-tg` |

**Tags:**

| Key | Value |
|-----|-------|
| `environment` | `production` |
| `tier` | `web` |
| `company` | `shoreline-systems` |

3. Click **Create load balancer**

---

## Step 10 — Verify the Web Tier

1. Wait for ALB status to show **Active**
2. Copy the ALB **DNS name**
3. Paste it in a browser

You should see: **shoreline solutions - Client Portal**

Traffic path confirmed: Internet → ALB → Web EC2 → Apache

---

## Step 11 — Launch the App Server EC2

1. Go to **EC2 → Instances → Launch instances**

| Setting | Value |
|---------|-------|
| Name | `shoreline-app-server-01` |
| AMI | Amazon Linux 2023 |
| Instance type | `t2.micro` |
| Key pair | Create new — `shoreline-app-kp` (RSA, .pem) — download and save it |
| VPC | `shoreline solutions-vpc` |
| Subnet | `shoreline solutions-subnet-private2-us-east-1b` |
| Auto-assign public IP | **Disabled** |
| Security group | `shoreline-app-ec2-sg` |
| Storage | 8 GiB gp3, encryption enabled |

**Tags:**

| Key | Value |
|-----|-------|
| `environment` | `production` |
| `tier` | `app` |
| `company` | `shoreline-systems` |

2. Click **Launch instance**

---

## Step 12 — Create the DB Subnet Group

1. Go to **RDS → Subnet groups → Create DB subnet group**

| Setting | Value |
|---------|-------|
| Name | `shoreline-db-subnet-group` |
| Description | `Private subnets for Shoreline RDS` |
| VPC | `shoreline solutions-vpc` |
| Subnets | Both private subnets (AZ1 + AZ2) |

2. Click **Create**

---

## Step 13 — Launch the RDS Instance

1. Go to **RDS → Databases → Create database**

| Setting | Value |
|---------|-------|
| Creation method | Full configuration |
| Engine | MySQL 8.0 |
| Template | Dev/Test |
| Deployment | Single DB instance |
| DB identifier | `shoreline-db-01` |
| Master username | `admin` |
| Credentials | Self managed — set a password and save it |
| Instance class | `db.t3.micro` |
| Storage | 20 GiB gp2 |
| VPC | `shoreline solutions-vpc` |
| Subnet group | `shoreline-db-subnet-group` |
| Public access | No |
| Security group | `shoreline-rds-sg` (remove default) |
| Initial database name | `shoreline` |
| Encryption | Enabled (default AWS key) |
| Automated backups | Enabled, 7 days |
| Deletion protection | Enabled |

**Tags:**

| Key | Value |
|-----|-------|
| `environment` | `production` |
| `tier` | `db` |
| `company` | `shoreline-systems` |

2. Click **Create database**

RDS takes 5–10 minutes to provision.

> **Cost note:** A `db.t3.micro` Single-AZ instance runs ~$0.50/day.
> Delete it when done recording. This guide has every setting — rebuilding takes 10 minutes.

---

## Environment Complete

All infrastructure is running. Here is what you have:

| Resource | Name | Status |
|----------|------|--------|
| VPC | shoreline solutions-vpc | Active |
| Web Server | shoreline-web-server-01 | Running |
| App Server | shoreline-app-server-01 | Running |
| Load Balancer | shoreline-alb-prod-web-east | Active |
| Database | shoreline-db-01 | Available |

**Next step:** Follow the [Security Services Setup Guide](shoreline-security-setup.md) to add the audit logging and threat detection layer on top of this environment.
