---
layout: default
title: DevOps Playbook
has_children: true
nav_order: 6
---

# DevOps Playbook: Infrastructure & HIPAA Compliance

> Secure AWS infrastructure configuration for HIPAA-compliant healthcare applications

---

## DevOps Responsibilities in HIPAA

Infrastructure teams are critical to HIPAA compliance. You're responsible for:

**Security Foundation:**
- Network architecture and segmentation
- Encryption at rest and in transit
- Access control and IAM policies
- Secure configuration of all services

**Monitoring & Response:**
- Audit logging and centralized log management
- Security monitoring and threat detection
- Incident response procedures
- Compliance monitoring

**Business Continuity:**
- Automated backups and retention
- Disaster recovery planning
- High availability architecture
- Recovery testing

---

## AWS Shared Responsibility Model

HIPAA compliance on AWS is **shared** between you and AWS:

### AWS Responsibilities ("Security OF the Cloud")
- Physical data center security
- Hardware and network infrastructure
- Managed service security (RDS engine patches, etc.)
- HIPAA-eligible service compliance

### Your Responsibilities ("Security IN the Cloud")
- IAM access control and MFA
- Data encryption configuration
- Network architecture and security groups
- Application security and code
- Audit logging and monitoring
- Backup and disaster recovery
- BAA activation and service selection

**Critical:** AWS provides HIPAA-eligible infrastructure, but YOU must configure it correctly.

---

## Our Tech Stack

This playbook focuses on:

**Compute:**
- Amazon ECS (Fargate) - Containerized applications
- EC2 (if needed) - Virtual machines

**Database:**
- Amazon RDS PostgreSQL - Encrypted, managed database

**Storage:**
- Amazon S3 - Encrypted object storage for backups

**Networking:**
- Amazon VPC - Isolated network environment
- Application Load Balancer - HTTPS termination
- NAT Gateway - Outbound internet access

**Security:**
- AWS KMS - Encryption key management
- AWS Secrets Manager - Credential storage
- IAM - Access control

**Monitoring:**
- CloudWatch Logs - Centralized logging
- CloudTrail - API audit trail
- GuardDuty - Threat detection
- AWS Config - Compliance monitoring

---

## Key Infrastructure Principles

### 1. Least Privilege Access
Grant minimum permissions required for each role/service.

### 2. Defense in Depth
Multiple layers of security (network, application, data).

### 3. Network Segmentation
Isolate PHI systems from public internet and non-compliant services.

### 4. Encryption Everywhere
All data encrypted at rest (KMS) and in transit (TLS 1.2+).

### 5. Audit Everything
Comprehensive logging of all API calls, access, and changes.

### 6. Automate Compliance
Use AWS Config Rules and automated checks.

---

## Quick Reference Checklist

Before deploying any HIPAA workload:

- [ ] AWS BAA signed via Artifact
- [ ] MFA enabled on all accounts
- [ ] VPC with private subnets configured
- [ ] RDS encryption enabled with KMS
- [ ] S3 buckets encrypted and access blocked
- [ ] Security groups follow least privilege
- [ ] CloudTrail enabled in all regions
- [ ] CloudWatch Logs retention ≥ 6 years
- [ ] AWS Config enabled with HIPAA rules
- [ ] Backup policies configured and tested

---

## Playbook Sections

### [1. AWS Setup](01-aws-setup.md)
AWS account configuration, BAA signing, IAM best practices

### [2. Infrastructure](02-infrastructure.md)
VPC, ECS, RDS, S3 - secure configuration patterns

### [3. Monitoring & Logging](03-monitoring.md)
CloudWatch, CloudTrail, GuardDuty, Config

### [4. Backup & Disaster Recovery](04-backup-dr.md)
RDS backups, S3 versioning, DR procedures

### [5. DevOps Checklist](05-checklist.md)
Complete implementation checklist

---

## HIPAA Requirements for Infrastructure

### Administrative Safeguards
- Formal security policies and procedures
- Risk assessment (annual, per 2025 rules)
- Workforce training and authorization
- Incident response plan

### Physical Safeguards
- Facility access controls (AWS responsibility)
- Workstation security (your laptops/devices)
- Device and media controls (proper disposal)

### Technical Safeguards
- **Access Control** - Unique user IDs, MFA, session timeouts
- **Audit Controls** - Log all access to ePHI
- **Integrity** - Mechanisms to ensure data isn't altered
- **Transmission Security** - Encryption in transit (TLS 1.2+)

---

## Common Mistakes to Avoid

**❌ Using non-HIPAA-eligible services for PHI**
→ Check [AWS HIPAA Eligible Services](https://aws.amazon.com/compliance/hipaa-eligible-services-reference/) list

**❌ Not signing the BAA before development**
→ Sign via AWS Artifact immediately

**❌ Public S3 buckets or RDS instances**
→ Always private subnets, block public access

**❌ Unencrypted data stores**
→ Enable KMS encryption for RDS, S3, EBS

**❌ Insufficient logging retention**
→ Minimum 6 years for audit logs

**❌ Overly permissive security groups**
→ Specific IP ranges and ports only

**❌ No MFA on privileged accounts**
→ Required per 2025 Security Rule updates

---

## Next Steps

1. **New to AWS + HIPAA?** Start with [AWS Setup](01-aws-setup.md)
2. **Building infrastructure?** Follow [Infrastructure](02-infrastructure.md)
3. **Setting up monitoring?** See [Monitoring](03-monitoring.md)
4. **Planning DR?** Read [Backup & DR](04-backup-dr.md)
5. **Pre-deployment check?** Use [DevOps Checklist](05-checklist.md)

---

_Last Updated: November 2025_
_Next: [AWS Setup →](01-aws-setup.md)_
