---
layout: default
title: Infrastructure
parent: DevOps Playbook
nav_order: 2
---

# Secure AWS Infrastructure for HIPAA

> Network architecture, ECS, RDS, and S3 configuration for PHI workloads

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    AWS Region (us-east-1)                   │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐ │
│  │                 VPC (10.0.0.0/16)                      │ │
│  │                                                        │ │
│  │  ┌──────────────────┐    ┌──────────────────┐        │ │
│  │  │ Public Subnet    │    │ Public Subnet    │        │ │
│  │  │ (10.0.1.0/24)    │    │ (10.0.2.0/24)    │        │ │
│  │  │  AZ-1            │    │  AZ-2            │        │ │
│  │  │                  │    │                  │        │ │
│  │  │ ┌──────────────┐ │    │                  │        │ │
│  │  │ │     ALB      │ │    │                  │        │ │
│  │  │ │ (HTTPS)      │ │    │                  │        │ │
│  │  │ └──────┬───────┘ │    │                  │        │ │
│  │  └────────┼──────────┘    └──────────────────┘        │ │
│  │           │                                            │ │
│  │  ┌────────┼──────────┐    ┌──────────────────┐        │ │
│  │  │ Private│Subnet    │    │ Private Subnet   │        │ │
│  │  │ (10.0.11.0/24)   │    │ (10.0.12.0/24)   │        │ │
│  │  │  AZ-1 │          │    │  AZ-2            │        │ │
│  │  │       ▼          │    │                  │        │ │
│  │  │ ┌────────────┐   │    │ ┌────────────┐   │        │ │
│  │  │ │ ECS Tasks  │   │    │ │ ECS Tasks  │   │        │ │
│  │  │ │ (Fargate)  │───┼────┼─┤ (Fargate)  │   │        │ │
│  │  │ └─────┬──────┘   │    │ └─────┬──────┘   │        │ │
│  │  └───────┼──────────┘    └───────┼──────────┘        │ │
│  │          │                       │                    │ │
│  │  ┌───────┼───────────────────────┼──────────┐        │ │
│  │  │ Data Subnet (10.0.21.0/24)    │          │        │ │
│  │  │       ▼                       ▼          │        │ │
│  │  │ ┌──────────────────────────────────┐    │        │ │
│  │  │ │   RDS PostgreSQL (Multi-AZ)      │    │        │ │
│  │  │ │   (Encrypted with KMS)           │    │        │ │
│  │  │ └──────────────────────────────────┘    │        │ │
│  │  └─────────────────────────────────────────┘        │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │         S3 Buckets (Private, Encrypted)            │ │
│  │  - Backups                                         │ │
│  │  - CloudTrail Logs                                 │ │
│  │  - Application Data                                │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │         AWS KMS (Encryption Keys)                  │ │
│  │  - RDS encryption key                              │ │
│  │  - S3 encryption key                               │ │
│  │  - EBS encryption key                              │ │
│  └────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

---

## 1. Network Architecture (VPC)

### VPC Design Principles

**Requirements:**
- Isolated network per environment (dev/staging/prod)
- Multi-AZ for high availability
- Public subnets ONLY for load balancers
- Private subnets for all PHI workloads
- Network segmentation (2025 HIPAA requirement)

### VPC Configuration

**CIDR Block:**
```
VPC: 10.0.0.0/16 (65,536 IPs)
├── Public Subnet AZ-1:  10.0.1.0/24
├── Public Subnet AZ-2:  10.0.2.0/24
├── Private Subnet AZ-1: 10.0.11.0/24
├── Private Subnet AZ-2: 10.0.12.0/24
├── Data Subnet AZ-1:    10.0.21.0/24
└── Data Subnet AZ-2:    10.0.22.0/24
```

**Subnet Types:**

| Subnet Type | Purpose | Internet Access | NAT Gateway |
|-------------|---------|-----------------|-------------|
| Public | ALB, NAT Gateway | Yes (IGW) | N/A |
| Private | ECS tasks, EC2 | Outbound only | Yes |
| Data | RDS, ElastiCache | No | No |

### Internet Gateway & NAT Gateway

**Internet Gateway (IGW):**
- Attached to VPC
- Only public subnets route to IGW
- Used for ALB inbound traffic

**NAT Gateway:**
- Deployed in **each** public subnet (one per AZ)
- Allows private subnets outbound internet (for updates, API calls)
- Private subnets route `0.0.0.0/0` → NAT Gateway

**Cost Optimization:** Use single NAT Gateway (not HA) for dev/staging.

### Route Tables

**Public Subnet Route Table:**
```
Destination       Target
10.0.0.0/16       local
0.0.0.0/0         igw-xxxxx
```

**Private Subnet Route Table:**
```
Destination       Target
10.0.0.0/16       local
0.0.0.0/0         nat-xxxxx (NAT Gateway in public subnet)
```

**Data Subnet Route Table:**
```
Destination       Target
10.0.0.0/16       local
```

---

## 2. Security Groups (Firewall Rules)

### Least Privilege Principle

**Rule:** Allow ONLY necessary traffic, deny everything else (implicit).

### Application Load Balancer (ALB) Security Group

```
Inbound:
- Port 443 (HTTPS) from 0.0.0.0/0 (public internet)

Outbound:
- Port 8080 (or app port) to ECS Security Group
```

### ECS Tasks Security Group

```
Inbound:
- Port 8080 from ALB Security Group
- Port 443 from VPC CIDR (for internal service-to-service)

Outbound:
- Port 5432 to RDS Security Group (PostgreSQL)
- Port 443 to 0.0.0.0/0 (HTTPS to external APIs)
```

### RDS Security Group

```
Inbound:
- Port 5432 from ECS Security Group ONLY

Outbound:
- None (database should not initiate outbound)
```

**Critical:** Never allow `0.0.0.0/0` (all IPs) to RDS.

### Network ACLs (Optional)

Use for additional defense-in-depth:
- Block known malicious IP ranges
- Deny traffic between subnets (data subnet isolated)

---

## 3. ECS (Elastic Container Service)

### ECS Launch Type: Fargate

**Why Fargate:**
- Serverless (no EC2 management)
- Automatic patching
- Pay-per-use
- Easier compliance (AWS manages underlying hosts)

**Alternative:** ECS on EC2 (more control, more management overhead).

### Task Definition Security

**Example Secure Task Definition:**

```json
{
  "family": "hipaa-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "taskRoleArn": "arn:aws:iam::ACCOUNT:role/ecsTaskRole",
  "executionRoleArn": "arn:aws:iam::ACCOUNT:role/ecsExecutionRole",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/app:latest",
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/hipaa-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:ACCOUNT:secret:db-password"
        }
      ],
      "environment": [
        {"name": "NODE_ENV", "value": "production"}
      ]
    }
  ]
}
```

**Key Points:**
- `taskRoleArn`: Permissions for app to access AWS services
- `executionRoleArn`: Permissions for ECS to pull images, write logs
- `secrets`: Use **Secrets Manager** (not environment variables for sensitive data)
- `logConfiguration`: Send logs to CloudWatch

### Secrets Management

**❌ Never:**
- Hardcode credentials in code
- Store in environment variables (plaintext)
- Commit secrets to git

**✅ Use AWS Secrets Manager:**

```bash
# Store database password
aws secretsmanager create-secret \
  --name prod/db/password \
  --secret-string "your-secure-password" \
  --kms-key-id alias/aws/secretsmanager

# Reference in ECS task definition (see above)
```

**Benefits:**
- Automatic encryption (KMS)
- Automatic rotation support
- Audit trail via CloudTrail
- Fine-grained IAM permissions

### Container Image Security

**Best Practices:**
- Scan images for vulnerabilities (ECR image scanning)
- Use minimal base images (alpine, distroless)
- Don't run as root user
- Keep images updated

```dockerfile
# Example secure Dockerfile
FROM node:20-alpine

# Don't run as root
RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001
USER nodejs

WORKDIR /app
COPY --chown=nodejs:nodejs . .
RUN npm ci --only=production

EXPOSE 8080
CMD ["node", "dist/main.js"]
```

---

## 4. RDS PostgreSQL

### Encryption Configuration

**Encryption at Rest (Required):**

```bash
# Enable when creating RDS instance
--storage-encrypted \
--kms-key-id arn:aws:kms:us-east-1:ACCOUNT:key/xxxxx
```

**Encryption in Transit (Required):**

```bash
# PostgreSQL parameter group
rds.force_ssl = 1

# Application connection string
postgresql://user:pass@host:5432/db?sslmode=require
```

### Network Isolation

**Critical Rules:**
- Deploy in **private data subnets**
- No public accessibility (`--no-publicly-accessible`)
- Security group allows ONLY ECS security group

```bash
aws rds create-db-instance \
  --db-instance-identifier hipaa-db \
  --db-instance-class db.t3.medium \
  --engine postgres \
  --engine-version 16.1 \
  --master-username dbadmin \
  --master-user-password <from-secrets-manager> \
  --allocated-storage 100 \
  --storage-type gp3 \
  --storage-encrypted \
  --kms-key-id alias/rds-encryption \
  --vpc-security-group-ids sg-xxxxx \
  --db-subnet-group-name data-subnet-group \
  --no-publicly-accessible \
  --backup-retention-period 30 \
  --preferred-backup-window "03:00-04:00" \
  --multi-az \
  --auto-minor-version-upgrade \
  --enabled-cloudwatch-logs-exports '["postgresql"]'
```

### RDS Parameter Group

**Required Parameters:**

```
# Force SSL connections
rds.force_ssl = 1

# Audit logging (pgaudit extension)
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'all'
pgaudit.log_catalog = on
pgaudit.log_parameter = on

# Connection logging
log_connections = 1
log_disconnections = 1
```

### Multi-AZ Deployment

**Why Multi-AZ:**
- Automatic failover (< 2 min)
- Synchronous replication
- No data loss
- HIPAA best practice

**Cost:** ~2x single AZ, but required for production.

### Backup Configuration

```bash
# Automated backups
--backup-retention-period 30  # days (use 7+ years for compliance)
--preferred-backup-window "03:00-04:00"  # UTC

# Manual snapshots (for major changes)
aws rds create-db-snapshot \
  --db-instance-identifier hipaa-db \
  --db-snapshot-identifier before-migration-2025-11-26
```

---

## 5. S3 Bucket Security

### Bucket Configuration

**Critical Settings:**

```bash
# Create bucket
aws s3api create-bucket \
  --bucket hipaa-app-backups-prod \
  --region us-east-1

# Block all public access (REQUIRED)
aws s3api put-public-access-block \
  --bucket hipaa-app-backups-prod \
  --public-access-block-configuration \
    BlockPublicAcls=true,\
    IgnorePublicAcls=true,\
    BlockPublicPolicy=true,\
    RestrictPublicBuckets=true

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket hipaa-app-backups-prod \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:ACCOUNT:key/xxxxx"
      }
    }]
  }'

# Enable versioning (recommended)
aws s3api put-bucket-versioning \
  --bucket hipaa-app-backups-prod \
  --versioning-configuration Status=Enabled

# Enable access logging
aws s3api put-bucket-logging \
  --bucket hipaa-app-backups-prod \
  --bucket-logging-status '{
    "LoggingEnabled": {
      "TargetBucket": "hipaa-logs-bucket",
      "TargetPrefix": "s3-access-logs/"
    }
  }'
```

### Bucket Policy (Deny Unencrypted Uploads)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedObjectUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::hipaa-app-backups-prod/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    }
  ]
}
```

### Lifecycle Policies

```json
{
  "Rules": [
    {
      "Id": "Archive old backups to Glacier",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        }
      ],
      "Expiration": {
        "Days": 2555
      }
    }
  ]
}
```

**Retention:** 2555 days = 7 years (HIPAA recommendation).

---

## 6. AWS KMS (Key Management)

### Key Policies

**Principle:** Separate keys per service.

**Keys to Create:**
- `alias/rds-encryption` - RDS databases
- `alias/s3-encryption` - S3 buckets
- `alias/ebs-encryption` - EBS volumes

**Example Key Policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Enable IAM policies",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT:root"
      },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "Allow RDS to use key",
      "Effect": "Allow",
      "Principal": {
        "Service": "rds.amazonaws.com"
      },
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "*"
    }
  ]
}
```

### Key Rotation

**Automatic Rotation (Recommended):**

```bash
aws kms enable-key-rotation --key-id alias/rds-encryption
```

**Rotation Frequency:** Automatic rotation every 365 days.

---

## 7. Application Load Balancer (ALB)

### SSL/TLS Configuration

**Requirements:**
- TLS 1.2 minimum (TLS 1.3 preferred)
- Strong cipher suites
- Valid SSL certificate

**Certificate Management (AWS ACM):**

```bash
# Request certificate (DNS validation recommended)
aws acm request-certificate \
  --domain-name app.example.com \
  --validation-method DNS \
  --subject-alternative-names "*.example.com"
```

### Security Policy

```bash
# Use secure policy
--ssl-policy ELBSecurityPolicy-TLS13-1-2-2021-06
```

**Policy details:**
- TLS 1.2 and 1.3 only
- Strong ciphers (AES-GCM, ChaCha20)
- Perfect forward secrecy

### HTTP to HTTPS Redirect

```bash
# Listener rule: redirect HTTP → HTTPS
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=redirect,RedirectConfig='{
    Protocol=HTTPS,Port=443,StatusCode=HTTP_301
  }'
```

---

## Configuration Checklist

**VPC & Networking:**
- [ ] VPC created with private subnets
- [ ] Multi-AZ configuration (2+ AZs)
- [ ] NAT Gateway for outbound traffic
- [ ] No PHI resources in public subnets
- [ ] Security groups follow least privilege
- [ ] Network ACLs configured (optional)

**ECS Configuration:**
- [ ] Fargate launch type
- [ ] Task roles with least privilege
- [ ] Secrets stored in Secrets Manager
- [ ] Container logs sent to CloudWatch
- [ ] Image scanning enabled
- [ ] Tasks deployed in private subnets

**RDS Configuration:**
- [ ] Encryption at rest enabled (KMS)
- [ ] SSL/TLS required for connections
- [ ] Deployed in private data subnets
- [ ] Not publicly accessible
- [ ] Multi-AZ enabled
- [ ] Automated backups configured (30+ days)
- [ ] CloudWatch Logs enabled
- [ ] Parameter group configured (force_ssl)

**S3 Configuration:**
- [ ] All public access blocked
- [ ] Encryption enabled (KMS)
- [ ] Versioning enabled
- [ ] Access logging enabled
- [ ] Lifecycle policies configured
- [ ] Bucket policy denies unencrypted uploads

**KMS:**
- [ ] Separate keys per service
- [ ] Key rotation enabled
- [ ] Key policies restrict access
- [ ] CloudTrail logging key usage

**Load Balancer:**
- [ ] TLS 1.2+ only
- [ ] Valid SSL certificate (ACM)
- [ ] HTTP → HTTPS redirect
- [ ] Access logs enabled

---

## Next Steps

1. **Set up monitoring:** [CloudWatch, CloudTrail, GuardDuty →](03-monitoring.md)
2. **Configure backups:** [Backup and DR procedures →](04-backup-dr.md)
3. **Final checklist:** [DevOps compliance checklist →](05-checklist.md)

---

_Last Updated: November 2025_
_Next: [Monitoring & Logging →](03-monitoring.md)_
