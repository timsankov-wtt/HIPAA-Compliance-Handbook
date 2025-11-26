---
layout: default
title: AWS Setup
parent: DevOps Playbook
nav_order: 1
---

# AWS Setup for HIPAA Compliance

> Foundation: Configuring your AWS environment before handling any PHI

---

## Prerequisites

Before storing or processing PHI on AWS:

1. **AWS Business Account** (not personal/free tier)
2. **Business Associate Agreement signed** (via AWS Artifact)
3. **MFA enabled** on root and all privileged accounts
4. **Understanding of HIPAA requirements** ([HIPAA Basics](../01-fundamentals/01-hipaa-basics.md))

---

## Step 1: AWS Account Setup

### Account Type Requirements

**✅ Required:**
- AWS Business or Enterprise Support plan (recommended for production)
- Valid business contact information
- Dedicated AWS Organization (for multi-account strategy)

**❌ Not Suitable:**
- AWS Free Tier (for production PHI workloads)
- Personal accounts
- Shared development accounts handling PHI

### Root Account Security

Immediately after account creation:

```bash
# Root account checklist
- [ ] Enable MFA on root account (hardware token recommended)
- [ ] Create strong password (16+ characters)
- [ ] Delete root access keys (if any exist)
- [ ] Set up alternate contacts (security, billing, operations)
- [ ] Enable CloudTrail in all regions
```

**Critical:** Never use root account for daily operations. Lock it down immediately.

---

## Step 2: Signing the AWS BAA

### What is the AWS BAA?

The **Business Associate Agreement** is a legal contract required under HIPAA when AWS handles PHI on your behalf. Without a signed BAA:
- You **cannot** legally store PHI on AWS
- Even HIPAA-eligible services are not compliant

### How to Sign the BAA

**Via AWS Artifact (Recommended):**

1. Log into AWS Console
2. Navigate to **AWS Artifact** service
3. Go to **Agreements** → **Organization agreements** (for org-wide) or **Account agreements**
4. Find **AWS Business Associate Addendum**
5. Review the terms
6. Click **Accept and download**
7. **Save the signed BAA** (store securely, you'll need it for audits)

**Timeline:** BAA is effective immediately after acceptance.

### Organization-Wide vs Single Account

**Organization-Wide BAA:**
- ✅ Covers all accounts in AWS Organization
- ✅ Easier management for multi-account setups
- ✅ Recommended for production environments

**Single Account BAA:**
- Covers only one AWS account
- Must sign separately for each account
- Suitable for small projects

**Our Recommendation:** Use Organization-Wide BAA even if starting with one account.

---

## Step 3: HIPAA-Eligible Services

### Understanding Service Eligibility

Not all AWS services can handle PHI. As of **November 2025**, AWS offers **166+ HIPAA-eligible services**.

**Key Services for Our Stack:**

| Service | HIPAA Eligible | Use Case |
|---------|---------------|----------|
| Amazon ECS | ✅ Yes | Container orchestration |
| Amazon RDS (PostgreSQL) | ✅ Yes | Managed database |
| Amazon S3 | ✅ Yes | Object storage, backups |
| AWS KMS | ✅ Yes | Encryption key management |
| CloudWatch Logs | ✅ Yes | Centralized logging |
| CloudTrail | ✅ Yes | API audit trail |
| AWS Secrets Manager | ✅ Yes | Credential storage |
| Application Load Balancer | ✅ Yes | HTTPS termination |
| Amazon VPC | ✅ Yes | Network isolation |

### Services to AVOID for PHI

**❌ Not HIPAA-Eligible (examples):**
- Amazon Lightsail
- AWS Amplify (some features)
- Amazon WorkMail (certain configurations)

**⚠️ Eligible but Restricted:**
- Amazon SageMaker (excludes Studio Lab, Ground Truth Plus)
- Amazon Pinpoint (excludes Voice, WhatsApp)
- Amazon Comprehend Medical (requires BAA)

**Official List:** [AWS HIPAA Eligible Services Reference](https://aws.amazon.com/compliance/hipaa-eligible-services-reference/)

### Critical Rule

**Only store/process PHI using HIPAA-eligible services AND after signing the BAA.**

---

## Step 4: IAM Best Practices

### Root Account Protection

```bash
# NEVER use root account except for:
- Changing account settings
- Closing the account
- Restoring IAM user permissions (emergency)
```

### User Management

**Create IAM Users with:**
- Unique credentials per person (no shared accounts)
- MFA enforcement (required per 2025 HIPAA updates)
- Least privilege access
- Access keys rotation policy (90 days)

**Example IAM Policy for Developers:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecs:DescribeTasks",
        "ecs:ListTasks",
        "rds:DescribeDBInstances",
        "logs:GetLogEvents"
      ],
      "Resource": "*",
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    }
  ]
}
```

### Service Roles

Use **IAM Roles** (not users) for services:
- ECS Task Roles (for container permissions)
- Lambda Execution Roles
- EC2 Instance Profiles

**Benefits:**
- No long-term credentials
- Automatic credential rotation
- Fine-grained permissions per service

### MFA Enforcement

**2025 Requirement:** MFA mandatory for remote access to ePHI systems.

**Enforce MFA:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

Attach this policy to all users/groups accessing PHI resources.

---

## Step 5: Multi-Account Strategy (Optional)

### Why Multi-Account?

**Benefits:**
- **Isolation:** Dev/staging/prod separation
- **Blast radius reduction:** Breaches contained to one account
- **Cost tracking:** Clear cost allocation
- **Compliance:** Easier audit scoping

### Architecture Example

```
AWS Organization (BAA covers all)
├── Management Account (billing, IAM Identity Center)
├── Security Account (CloudTrail logs, Config aggregation)
├── Dev Account (development, testing)
├── Staging Account (pre-production)
└── Production Account (PHI workloads)
```

### AWS Organizations Setup

1. Create **AWS Organization** in management account
2. Enable **all features** (not just consolidated billing)
3. Apply **Service Control Policies (SCPs)** to enforce security
4. Sign **organization-wide BAA** in AWS Artifact
5. Use **AWS IAM Identity Center** for SSO + MFA

**Example SCP (Block Public S3 Buckets):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": [
        "s3:PutAccountPublicAccessBlock"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "s3:AllowPublicAccess": "false"
        }
      }
    }
  ]
}
```

---

## Step 6: Enable Core Security Services

### CloudTrail (Required)

**Purpose:** Log all API calls for audit trail.

**Setup:**
```bash
# Enable CloudTrail in ALL regions
- [ ] Create trail in management account
- [ ] Enable log file validation
- [ ] Store logs in dedicated S3 bucket (encrypted)
- [ ] Set retention: 7+ years
- [ ] Enable multi-region logging
```

### AWS Config (Recommended)

**Purpose:** Track resource configuration changes and compliance.

**Setup:**
```bash
- [ ] Enable AWS Config in all regions
- [ ] Configure Config Rules for HIPAA compliance
- [ ] Set up SNS notifications for non-compliance
- [ ] Aggregate findings to security account (if multi-account)
```

**Example Config Rule:** Require RDS encryption.

### GuardDuty (Recommended)

**Purpose:** Threat detection and monitoring.

**Setup:**
```bash
- [ ] Enable GuardDuty in all regions
- [ ] Configure finding export to S3
- [ ] Set up SNS alerts for high-severity findings
```

---

## Step 7: Initial Configuration Checklist

### Before Any Development

**Account Setup:**
- [ ] AWS Business Account created
- [ ] Root account MFA enabled
- [ ] Root access keys deleted
- [ ] AWS Support plan activated

**BAA & Compliance:**
- [ ] AWS BAA signed via Artifact
- [ ] BAA document saved securely
- [ ] HIPAA-eligible services list reviewed
- [ ] Only eligible services approved for use

**IAM & Access:**
- [ ] IAM users created (no root usage)
- [ ] MFA enforced on all users
- [ ] Service roles configured
- [ ] Password policy set (12+ chars, rotation)
- [ ] Access key rotation policy (90 days)

**Logging & Monitoring:**
- [ ] CloudTrail enabled (all regions)
- [ ] CloudWatch Logs configured
- [ ] AWS Config enabled
- [ ] GuardDuty activated
- [ ] Log retention ≥ 6 years

**Network Foundation:**
- [ ] VPC created with private subnets
- [ ] Security groups configured (least privilege)
- [ ] No public-facing databases/storage

---

## Common Mistakes

**❌ Starting development before BAA**
→ Sign BAA on day one, before any PHI touches AWS

**❌ Using non-eligible services**
→ Check official list, don't assume eligibility

**❌ Weak IAM permissions**
→ Use least privilege, enforce MFA

**❌ Missing CloudTrail**
→ Required for audit compliance

**❌ Public RDS/S3 resources**
→ Always private subnets, block public access

---

## Validation

Test your setup:

```bash
# Verify BAA is signed
aws artifact get-agreements --region us-east-1

# Check CloudTrail status
aws cloudtrail get-trail-status --name <trail-name>

# Verify MFA enforcement
aws iam get-account-summary | grep MFADevices

# List HIPAA-eligible services in use
aws config describe-compliance-by-config-rule
```

---

## Next Steps

1. **Infrastructure Setup:** [Configure VPC, ECS, RDS →](02-infrastructure.md)
2. **Monitoring:** [Set up CloudWatch and alerts →](03-monitoring.md)
3. **Backups:** [Configure backup and DR →](04-backup-dr.md)

---

## Additional Resources

**Official AWS:**
- [AWS HIPAA Compliance](https://aws.amazon.com/compliance/hipaa-compliance/)
- [HIPAA Eligible Services](https://aws.amazon.com/compliance/hipaa-eligible-services-reference/)
- [AWS Artifact](https://aws.amazon.com/artifact/)

**Tools:**
- [AWS Well-Architected Tool](https://aws.amazon.com/well-architected-tool/)
- [Trusted Advisor](https://aws.amazon.com/premiumsupport/technology/trusted-advisor/)

---

_Last Updated: November 2025_
_Next: [Infrastructure Setup →](02-infrastructure.md)_
