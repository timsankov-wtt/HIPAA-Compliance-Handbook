---
layout: default
title: DevOps Checklist
parent: DevOps Playbook
nav_order: 5
---

# DevOps HIPAA Compliance Checklist

> Complete pre-deployment verification for infrastructure teams

---

## How to Use This Checklist

1. **Initial Setup:** Complete all "Critical" items before any PHI touches AWS
2. **Pre-Deployment:** Verify all items before production launch
3. **Ongoing:** Review "Maintenance" items quarterly
4. **Audits:** Use as evidence of compliance measures

**Status Tracking:**
- ✅ = Completed and verified
- 🔄 = In progress
- ⏸️ = Not applicable (document why)

---

## Critical (Must Have Before PHI)

### AWS Account Setup

- [ ] AWS Business Account created (not free tier)
- [ ] Business contact information configured
- [ ] AWS Support plan activated (Business or Enterprise)
- [ ] Root account MFA enabled (hardware token preferred)
- [ ] Root account password is strong (16+ characters)
- [ ] Root access keys deleted (or never created)
- [ ] Alternate contacts configured (security, billing, operations)

**Validation:** `aws iam get-account-summary` shows `AccountMFAEnabled: 1`

### Business Associate Agreement

- [ ] AWS BAA signed via AWS Artifact
- [ ] Organization-wide BAA (if using AWS Organizations)
- [ ] BAA document downloaded and stored securely
- [ ] All team members aware of BAA requirements
- [ ] Only HIPAA-eligible services approved for PHI use

**Link:** [AWS Artifact Console](https://console.aws.amazon.com/artifact/)

**Validation:** Check AWS Artifact agreements page

### IAM & Access Control

- [ ] IAM password policy enforced (12+ chars, complexity, 90-day rotation)
- [ ] MFA required for all users with console access
- [ ] MFA enforced via IAM policy (deny actions without MFA)
- [ ] No shared accounts (unique IAM user per person)
- [ ] Root account not used for daily operations
- [ ] Service roles created (not IAM users for applications)
- [ ] Access key rotation policy (90 days)
- [ ] Least privilege principle applied to all policies
- [ ] No wildcard (`*`) permissions in production policies
- [ ] IAM Access Analyzer enabled

**Validation:**
```bash
aws iam get-account-password-policy
aws iam list-users --query 'Users[?PasswordLastUsed==`null`]'
```

### Logging & Auditing (Required)

- [ ] CloudTrail enabled in all regions
- [ ] CloudTrail multi-region trail configured
- [ ] CloudTrail log file validation enabled
- [ ] CloudTrail logs stored in encrypted S3 bucket
- [ ] CloudTrail S3 bucket has deny delete policy
- [ ] CloudTrail retention ≥ 7 years
- [ ] CloudWatch Logs integration enabled
- [ ] VPC Flow Logs enabled for all VPCs
- [ ] Log retention configured (6+ years for audit logs)
- [ ] Logs encrypted with KMS

**Validation:**
```bash
aws cloudtrail describe-trails
aws cloudtrail get-trail-status --name <trail-name>
```

---

## Network & Infrastructure

### VPC Configuration

- [ ] VPC created with private subnets
- [ ] Multi-AZ deployment (minimum 2 availability zones)
- [ ] Public subnets only for load balancers/NAT gateways
- [ ] Private subnets for application workloads
- [ ] Data subnets isolated for databases (no internet access)
- [ ] NAT Gateway configured for outbound traffic
- [ ] Internet Gateway only attached to public subnets
- [ ] Network segmentation implemented (2025 requirement)
- [ ] No default VPC used for production

**Validation:** Review VPC route tables and subnet configurations

### Security Groups

- [ ] Default security groups have no rules
- [ ] All security groups follow least privilege
- [ ] No `0.0.0.0/0` access to RDS or data stores
- [ ] Specific port/protocol rules (no broad ranges)
- [ ] Security groups use descriptive names
- [ ] Inbound rules documented and justified
- [ ] Regular security group audits scheduled

**Validation:**
```bash
aws ec2 describe-security-groups --query 'SecurityGroups[?IpPermissions[?IpRanges[?CidrIp==`0.0.0.0/0`]]]'
```

### Load Balancer Security

- [ ] Application Load Balancer in public subnets
- [ ] HTTPS listener configured (port 443)
- [ ] HTTP redirects to HTTPS (port 80 → 443)
- [ ] SSL certificate from AWS ACM
- [ ] TLS 1.2 minimum (TLS 1.3 preferred)
- [ ] Security policy: `ELBSecurityPolicy-TLS13-1-2-2021-06` or stronger
- [ ] Access logs enabled
- [ ] Access logs stored in encrypted S3 bucket

**Validation:** Check ALB listeners and SSL policies

---

## Database Security (RDS PostgreSQL)

### Encryption

- [ ] Encryption at rest enabled (AWS KMS)
- [ ] Dedicated KMS key for RDS (not default key)
- [ ] KMS key rotation enabled
- [ ] SSL/TLS required for database connections (`rds.force_ssl=1`)
- [ ] Application connects with `sslmode=require`

**Validation:**
```bash
aws rds describe-db-instances --query 'DBInstances[*].[DBInstanceIdentifier,StorageEncrypted,KmsKeyId]'
```

### Network & Access

- [ ] RDS in private data subnets
- [ ] Not publicly accessible (`PubliclyAccessible=false`)
- [ ] Security group allows only application security group
- [ ] No direct internet access to RDS
- [ ] Database port (5432) not exposed to `0.0.0.0/0`

**Validation:**
```bash
aws rds describe-db-instances --query 'DBInstances[?PubliclyAccessible==`true`]'
```

### Configuration

- [ ] Multi-AZ deployment enabled
- [ ] Automated backups enabled (30-day retention)
- [ ] Backup retention period ≥ 7 years (via manual snapshots)
- [ ] Preferred backup window configured
- [ ] Point-in-time recovery tested
- [ ] CloudWatch Logs enabled (postgresql logs)
- [ ] Enhanced monitoring enabled
- [ ] Deletion protection enabled
- [ ] Auto minor version upgrade enabled

**Parameter Group:**
- [ ] `rds.force_ssl = 1`
- [ ] `log_connections = 1`
- [ ] `log_disconnections = 1`
- [ ] `shared_preload_libraries = 'pgaudit'` (if using pgAudit)

### Backups

- [ ] Automated daily backups configured
- [ ] Manual snapshots for long-term retention
- [ ] Cross-region snapshots for DR (if applicable)
- [ ] Backup encryption verified
- [ ] Quarterly restore tests documented
- [ ] Backup tags include compliance metadata

---

## S3 Bucket Security

### Bucket Configuration

- [ ] All public access blocked (all 4 settings enabled)
- [ ] Server-side encryption enabled (SSE-KMS)
- [ ] Dedicated KMS key for S3 (not default)
- [ ] Bucket versioning enabled
- [ ] Lifecycle policies configured
- [ ] MFA delete enabled (optional, for critical buckets)
- [ ] Access logging enabled
- [ ] Object lock enabled (for immutable logs, optional)

**Validation:**
```bash
aws s3api get-public-access-block --bucket <bucket-name>
aws s3api get-bucket-encryption --bucket <bucket-name>
```

### Bucket Policies

- [ ] Deny unencrypted uploads
- [ ] Deny deletion of objects (for log buckets)
- [ ] Require specific KMS key for encryption
- [ ] Block public ACLs
- [ ] No overly permissive policies (`*` principal)

**Example Policy Check:**
```bash
aws s3api get-bucket-policy --bucket <bucket-name>
```

### Data Classification

- [ ] Buckets tagged with data classification
- [ ] Separate buckets for PHI vs non-PHI
- [ ] Bucket naming convention enforced
- [ ] Access reviewed quarterly

---

## ECS/Container Security

### Task Configuration

- [ ] Fargate launch type (or hardened EC2)
- [ ] Task role with least privilege permissions
- [ ] Execution role for ECS (pull images, write logs)
- [ ] Secrets stored in AWS Secrets Manager (not env vars)
- [ ] Secrets injected at runtime
- [ ] No hardcoded credentials in images or code
- [ ] Container logs sent to CloudWatch
- [ ] Tasks deployed in private subnets
- [ ] Security groups restrict task access

### Container Images

- [ ] Images stored in Amazon ECR
- [ ] Image scanning enabled (vulnerability detection)
- [ ] Images use minimal base (Alpine, distroless)
- [ ] Containers don't run as root user
- [ ] Images regularly updated
- [ ] No secrets in Dockerfile or layers
- [ ] Image tags are immutable (not `latest`)

**Validation:**
```bash
aws ecr describe-images --repository-name <repo-name> --query 'imageDetails[*].[imageTags,imageScanStatus]'
```

---

## Monitoring & Alerting

### CloudWatch

- [ ] Log groups for all services
- [ ] Log retention configured (2190+ days for audit logs)
- [ ] Logs encrypted with KMS
- [ ] Metric filters for critical events
- [ ] CloudWatch Insights queries saved
- [ ] Dashboards created for key metrics

**Critical Metric Filters:**
- [ ] Failed authentication attempts
- [ ] PHI access count
- [ ] Security group changes
- [ ] IAM policy changes
- [ ] RDS configuration changes

### Alarms

- [ ] High failed authentication rate alert
- [ ] RDS CPU > 80% alert
- [ ] RDS storage < 20% alert
- [ ] ECS task failure alert
- [ ] GuardDuty high-severity finding alert
- [ ] Config non-compliance alert

**SNS Topics:**
- [ ] Security alerts topic created
- [ ] Compliance alerts topic created
- [ ] Operational alerts topic created
- [ ] Subscriptions configured (email, Slack, PagerDuty)

### AWS Config

- [ ] AWS Config enabled in all regions
- [ ] HIPAA conformance pack deployed
- [ ] Config rules for encryption (RDS, S3, EBS)
- [ ] Config rules for public access (S3, RDS)
- [ ] Config rules for CloudTrail, VPC Flow Logs
- [ ] Non-compliance notifications configured
- [ ] Config history retained

**Key Rules:**
- [ ] `rds-storage-encrypted`
- [ ] `s3-bucket-server-side-encryption-enabled`
- [ ] `cloud-trail-enabled`
- [ ] `root-account-mfa-enabled`
- [ ] `vpc-flow-logs-enabled`
- [ ] `rds-instance-public-access-check`

### GuardDuty

- [ ] GuardDuty enabled in all regions
- [ ] Findings exported to S3
- [ ] High-severity findings → SNS alerts
- [ ] Findings reviewed weekly
- [ ] Threat intelligence feeds enabled

---

## Encryption & Key Management

### AWS KMS

- [ ] Separate KMS keys per service (RDS, S3, EBS, Secrets Manager)
- [ ] KMS keys use aliases (e.g., `alias/rds-encryption`)
- [ ] Key rotation enabled (automatic annual rotation)
- [ ] Key policies restrict access
- [ ] CloudTrail logs key usage
- [ ] No default AWS managed keys for PHI

**Validation:**
```bash
aws kms list-keys
aws kms get-key-rotation-status --key-id <key-id>
```

### Secrets Management

- [ ] Secrets stored in AWS Secrets Manager
- [ ] Secrets encrypted with KMS
- [ ] No secrets in code or environment variables
- [ ] Automatic secret rotation enabled (where supported)
- [ ] Access to secrets restricted via IAM
- [ ] Secrets Manager logs to CloudTrail

---

## Backup & Disaster Recovery

### RDS Backups

- [ ] Automated backups enabled (30-day retention)
- [ ] Manual snapshots for long-term (monthly/yearly)
- [ ] Backup window configured (low-traffic period)
- [ ] Cross-region snapshots (if DR required)
- [ ] Point-in-time recovery tested
- [ ] Backup encryption verified

### S3 Backups

- [ ] Versioning enabled
- [ ] Lifecycle policies configured
- [ ] Cross-region replication (if DR required)
- [ ] Versioning retention meets compliance (7 years)

### DR Procedures

- [ ] DR region identified
- [ ] DR runbook documented
- [ ] Infrastructure as Code for DR region
- [ ] Route53 health checks configured
- [ ] DR drill conducted annually
- [ ] RTO/RPO documented and meet requirements

### Testing

- [ ] Quarterly restore tests scheduled
- [ ] Test results documented
- [ ] Restore procedure documented
- [ ] Recovery time measured (RTO validation)

---

## Ongoing Maintenance (Important)

### Quarterly Tasks

- [ ] Backup restore test
- [ ] Access review (IAM users, roles)
- [ ] Security group audit
- [ ] Unused resources cleanup
- [ ] GuardDuty findings review
- [ ] Config compliance report
- [ ] Patch/update review

### Annual Tasks

- [ ] Formal risk assessment (2025 requirement)
- [ ] Vulnerability scan
- [ ] Disaster recovery drill
- [ ] BAA renewal review
- [ ] Compliance audit (internal or external)
- [ ] Policy and procedure review

### Continuous Monitoring

- [ ] CloudWatch alarms reviewed weekly
- [ ] GuardDuty findings triaged daily
- [ ] Config non-compliance resolved within 7 days
- [ ] Access logs reviewed monthly
- [ ] CloudTrail logs analyzed monthly

---

## Advanced/Optional

### Additional Security

- [ ] AWS Security Hub enabled
- [ ] AWS Inspector for vulnerability scanning
- [ ] AWS Macie for sensitive data discovery (S3)
- [ ] AWS Firewall Manager (multi-account)
- [ ] VPC endpoints for AWS services (private connectivity)
- [ ] AWS Systems Manager Session Manager (no SSH keys)

### Compliance Automation

- [ ] AWS Audit Manager for compliance reporting
- [ ] Automated compliance checks (CI/CD)
- [ ] Infrastructure as Code (Terraform/CloudFormation)
- [ ] Automated remediation (Lambda + Config)

### Multi-Account Strategy

- [ ] AWS Organizations configured
- [ ] Service Control Policies (SCPs) enforced
- [ ] Centralized logging account
- [ ] Separate accounts for dev/staging/prod
- [ ] AWS IAM Identity Center (SSO) for MFA

---

## Sign-Off

**Completed By:** ___________________________
**Date:** ___________________________
**Reviewed By:** ___________________________
**Next Review:** ___________________________

---

## Validation Commands

Run these commands to verify configuration:

```bash
# Check CloudTrail status
aws cloudtrail get-trail-status --name <trail-name>

# Check RDS encryption
aws rds describe-db-instances --query 'DBInstances[*].[DBInstanceIdentifier,StorageEncrypted]'

# Check S3 public access
aws s3api get-public-access-block --bucket <bucket-name>

# Check Config rules compliance
aws configservice describe-compliance-by-config-rule

# Check GuardDuty status
aws guardduty list-detectors

# Check IAM password policy
aws iam get-account-password-policy

# Check security groups with 0.0.0.0/0
aws ec2 describe-security-groups --query 'SecurityGroups[?IpPermissions[?IpRanges[?CidrIp==`0.0.0.0/0`]]].[GroupId,GroupName]'
```

---

## Related Resources

- [AWS Setup Guide](01-aws-setup.md)
- [Infrastructure Configuration](02-infrastructure.md)
- [Monitoring Setup](03-monitoring.md)
- [Backup & DR](04-backup-dr.md)

---

_Last Updated: November 2025_
_This checklist should be reviewed and updated quarterly._
