# shoreline solutions — Infrastructure Setup Guide

## Prerequisites

Before starting this guide, make sure you have the following in place:

- An AWS account created and accessible
- IAM user or role with administrative permissions (or at minimum: EC2, VPC, RDS, S3, IAM, Lambda, GuardDuty, Security Hub, SSM full access)
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

**Inbound rules:**

| Type | Port | Source |
|------|------|--------|
| HTTP | 80 | `shoreline-alb-sg-01` (security group ID) |
| HTTPS | 443 | `10.0.0.0/16` |

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

**Inbound rules:**

| Type | Port | Source |
|------|------|--------|
| HTTP | 80 | `shoreline-web-ec2-sg` (security group ID) |
| HTTPS | 443 | `10.0.0.0/16` |

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

## Step 6 — Create the IAM Role for EC2

This role allows SSM Session Manager to connect to EC2 instances without SSH.

1. Go to **IAM → Roles → Create role**
2. Trusted entity: **AWS Service**
3. Use case: **EC2**
4. Attach policy: `AmazonSSMManagedInstanceCore`
5. Role name: `shoreline-ec2-ssm-role`
6. Click **Create role**

---

## Step 7 — Launch the Web Server EC2

1. Go to **EC2 → Instances → Launch instances**

| Setting | Value |
|---------|-------|
| Name | `shoreline-web-server-01` |
| AMI | Amazon Linux 2023 |
| Instance type | `t2.micro` |
| Key pair | No key pair |
| VPC | `shoreline solutions-vpc` |
| Subnet | `shoreline solutions-subnet-private1-us-east-1a` |
| Auto-assign public IP | **Disabled** |
| Security group | `shoreline-web-ec2-sg` |
| IAM instance profile | `shoreline-ec2-ssm-role` |
| Storage | 8 GiB gp3, encryption enabled |

**Tags:**

| Key | Value |
|-----|-------|
| `environment` | `production` |
| `tier` | `web` |
| `company` | `shoreline-systems` |

2. Click **Launch instance**

---

## Step 8 — Create VPC Endpoints for SSM

The web and app servers are in private subnets with no internet route.
These endpoints allow SSM to reach the instances privately.

Go to **VPC → Endpoints → Create endpoint** and create all three below.

**Settings for all three:**

| Setting | Value |
|---------|-------|
| Type | AWS services |
| VPC | `shoreline solutions-vpc` |
| Subnets | Both private subnets |
| Security group | `shoreline-web-ec2-sg` |
| Private DNS | Enabled |

| Endpoint Name | Service |
|--------------|---------|
| `shoreline-ssm-endpoint` | `com.amazonaws.us-east-1.ssm` |
| `shoreline-ssmmessages-endpoint` | `com.amazonaws.us-east-1.ssmmessages` |
| `shoreline-ec2messages-endpoint` | `com.amazonaws.us-east-1.ec2messages` |

Wait 2–3 minutes after creation. If Session Manager shows Offline, stop and start the EC2.

---

## Step 9 — Install Apache on the Web Server

1. Go to **EC2 → Instances → shoreline-web-server-01**
2. Click **Connect → Session Manager → Connect**
3. Run these commands:

```bash
sudo yum update -y
sudo yum install -y httpd
sudo systemctl start httpd
sudo systemctl enable httpd
echo "<h1>shoreline solutions - Client Portal</h1>" | sudo tee /var/www/html/index.html
```

---

## Step 10 — Create the Target Group

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

## Step 11 — Create the Application Load Balancer

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

## Step 12 — Verify the Web Tier

1. Wait for ALB status to show **Active**
2. Copy the ALB **DNS name**
3. Paste it in a browser

You should see: **shoreline solutions - Client Portal**

Traffic path confirmed: Internet → ALB → Web EC2 → Apache

---

## Step 13 — Launch the App Server EC2

1. Go to **EC2 → Instances → Launch instances**

| Setting | Value |
|---------|-------|
| Name | `shoreline-app-server-01` |
| AMI | Amazon Linux 2023 |
| Instance type | `t2.micro` |
| Key pair | No key pair |
| VPC | `shoreline solutions-vpc` |
| Subnet | `shoreline solutions-subnet-private2-us-east-1b` |
| Auto-assign public IP | **Disabled** |
| Security group | `shoreline-app-ec2-sg` |
| IAM instance profile | `shoreline-ec2-ssm-role` |
| Storage | 8 GiB gp3, encryption enabled |

**Tags:**

| Key | Value |
|-----|-------|
| `environment` | `production` |
| `tier` | `app` |
| `company` | `shoreline-systems` |

2. Click **Launch instance**

---

## Step 14 — Create the DB Subnet Group

1. Go to **RDS → Subnet groups → Create DB subnet group**

| Setting | Value |
|---------|-------|
| Name | `shoreline-db-subnet-group` |
| Description | `Private subnets for Shoreline RDS` |
| VPC | `shoreline solutions-vpc` |
| Subnets | Both private subnets (AZ1 + AZ2) |

2. Click **Create**

---

## Step 15 — Launch the RDS Instance

1. Go to **RDS → Databases → Create database**

| Setting | Value |
|---------|-------|
| Creation method | Full configuration |
| Engine | MySQL 8.0 |
| Template | Dev/Test |
| Deployment | Single DB instance |
| DB identifier | `shoreline-db-01` |
| Master username | `admin` |
| Credentials | AWS Secrets Manager |
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


## Step 16 — Seed the Database (quick verify)

Goal: confirm the app server can reach RDS, and put one record in
place so the database isn't empty when we demo the security layer.

### Get endpoint + password
1. **RDS → shoreline-db-01 → Connectivity** → copy the **Endpoint**
2. **Secrets Manager** → secret for `shoreline-db-01` → **Retrieve secret value** → copy password

### Connect via SSM and seed
1. **EC2 → shoreline-app-server-01 → Connect → Session Manager**

```bash
sudo dnf install -y mariadb105
mysql -h <your-rds-endpoint> -u admin -p
```

```sql
USE shoreline;

CREATE TABLE shipments (
    shipment_id  VARCHAR(10) PRIMARY KEY,
    client_name  VARCHAR(100) NOT NULL,
    cargo_type   VARCHAR(50)  NOT NULL,
    status       VARCHAR(20)  NOT NULL
);

INSERT INTO shipments VALUES
('SL-1001', 'Harborview Imports', 'Electronics', 'In Transit');

SELECT * FROM shipments;
exit
```

One row returned = the full 3-tier path works and the DB holds live data.
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
