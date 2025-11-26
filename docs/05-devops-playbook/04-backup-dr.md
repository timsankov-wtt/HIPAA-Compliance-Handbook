---
layout: default
title: Backup & Disaster Recovery
parent: DevOps Playbook
nav_order: 4
---

# Backup & Disaster Recovery for HIPAA

> Ensuring data availability, integrity, and recoverability

---

## HIPAA Requirements

**Administrative Safeguards:**
- Data backup plan (§164.308(a)(7)(ii)(A))
- Disaster recovery plan (§164.308(a)(7)(ii)(B))
- Testing and revision procedures (§164.308(a)(7)(ii)(E))

**Key Mandates:**
- **Backups:** Regular, encrypted, and tested
- **Retention:** Minimum 6 years (varies by state, typically 6-7 years)
- **Encryption:** All backups must be encrypted
- **Testing:** Quarterly restore tests required (2025 update)

---

## Recovery Objectives

### RPO (Recovery Point Objective)

**Definition:** Maximum acceptable data loss (time between backups).

**Recommendations:**
- **Critical PHI systems:** 15 minutes - 1 hour
- **Standard systems:** 24 hours
- **Archival data:** 7 days

**Our Setup:**
- RDS automated backups: 5-minute point-in-time recovery
- ECS configuration: Infrastructure as Code (recreate in minutes)
- S3 data: Versioning (zero data loss)

### RTO (Recovery Time Objective)

**Definition:** Maximum acceptable downtime.

**Recommendations:**
- **Tier 1 (Critical):** < 1 hour
- **Tier 2 (Important):** < 4 hours
- **Tier 3 (Standard):** < 24 hours

**Our Setup:**
- Multi-AZ RDS: ~2 minutes automatic failover
- ECS Auto Scaling: 5-10 minutes for new tasks
- Full DR region: 1-4 hours (manual failover)

---

## 1. RDS Automated Backups

### Configuration

**Enable at database creation:**

```bash
aws rds create-db-instance \
  --db-instance-identifier hipaa-db \
  --backup-retention-period 30 \
  --preferred-backup-window "03:00-04:00" \
  --preferred-maintenance-window "Mon:04:00-Mon:05:00" \
  --copy-tags-to-snapshot \
  --deletion-protection
```

**Key Parameters:**

| Parameter | Value | Notes |
|-----------|-------|-------|
| `backup-retention-period` | 30 days | Use max 35 for automated (for long-term, use manual snapshots) |
| `preferred-backup-window` | 03:00-04:00 UTC | Low-traffic window |
| `copy-tags-to-snapshot` | true | Tag backups for compliance |
| `deletion-protection` | true | Prevent accidental deletion |

### Point-in-Time Recovery (PITR)

**Restore to any time** within retention period (down to 5-minute granularity).

```bash
# Restore to specific timestamp
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier hipaa-db \
  --target-db-instance-identifier hipaa-db-restored-2025-11-26 \
  --restore-time 2025-11-26T14:30:00Z \
  --vpc-security-group-ids sg-xxxxx \
  --db-subnet-group-name data-subnet-group
```

### Manual Snapshots (Long-Term Retention)

**Use for:**
- Before major migrations
- Monthly/yearly compliance snapshots
- Cross-region DR

```bash
# Create manual snapshot
aws rds create-db-snapshot \
  --db-instance-identifier hipaa-db \
  --db-snapshot-identifier hipaa-db-monthly-2025-11

# Manual snapshots are retained indefinitely until deleted
```

### Cross-Region Backup Replication

**For disaster recovery:**

```bash
# Copy snapshot to DR region
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier arn:aws:rds:us-east-1:ACCOUNT:snapshot:hipaa-db-monthly-2025-11 \
  --target-db-snapshot-identifier hipaa-db-monthly-2025-11-dr \
  --region us-west-2 \
  --kms-key-id arn:aws:kms:us-west-2:ACCOUNT:key/xxxxx
```

**Encrypted snapshots** require KMS key in target region.

---

## 2. S3 Backup Strategy

### Versioning

**Enable versioning** for all buckets containing PHI or critical data.

```bash
aws s3api put-bucket-versioning \
  --bucket hipaa-app-backups \
  --versioning-configuration Status=Enabled
```

**Benefits:**
- Protects against accidental deletion
- Enables recovery from application bugs
- Audit trail of object changes

**Cost:** Stores all versions (manage with lifecycle policies).

### Lifecycle Policies

**Transition old versions to cheaper storage:**

```json
{
  "Rules": [
    {
      "Id": "Archive old backups",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER_IR"
        },
        {
          "Days": 365,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ],
      "NoncurrentVersionTransitions": [
        {
          "NoncurrentDays": 30,
          "StorageClass": "GLACIER_IR"
        }
      ],
      "Expiration": {
        "Days": 2555
      },
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 90
      }
    }
  ]
}
```

**Explanation:**
- Day 0-30: Standard storage
- Day 30-90: Infrequent Access
- Day 90-365: Glacier Instant Retrieval
- Day 365+: Deep Archive (cheapest)
- Delete after 2555 days (7 years)
- Delete old versions after 90 days

**Cost Savings:** ~95% cheaper (Deep Archive vs Standard).

### Cross-Region Replication (CRR)

**For disaster recovery:**

```bash
# Enable replication
aws s3api put-bucket-replication \
  --bucket hipaa-app-backups \
  --replication-configuration '{
    "Role": "arn:aws:iam::ACCOUNT:role/S3ReplicationRole",
    "Rules": [
      {
        "Status": "Enabled",
        "Priority": 1,
        "Filter": {},
        "Destination": {
          "Bucket": "arn:aws:s3:::hipaa-app-backups-dr-us-west-2",
          "EncryptionConfiguration": {
            "ReplicaKmsKeyID": "arn:aws:kms:us-west-2:ACCOUNT:key/xxxxx"
          }
        }
      }
    ]
  }'
```

**Requirements:**
- Versioning enabled on both buckets
- IAM role with replication permissions
- Encryption keys in both regions

---

## 3. AWS Backup (Centralized Management)

### Why AWS Backup?

**Benefits:**
- Centralized backup policies
- Automated scheduling
- Cross-account/cross-region backups
- Compliance reporting
- Point-in-time recovery

**Supports:**
- RDS
- EBS volumes
- EFS file systems
- DynamoDB tables

### Backup Plan Example

```json
{
  "BackupPlanName": "HIPAA-Daily-Backup",
  "Rules": [
    {
      "RuleName": "DailyBackups",
      "TargetBackupVault": "HIPAA-Vault",
      "ScheduleExpression": "cron(0 3 * * ? *)",
      "StartWindowMinutes": 60,
      "CompletionWindowMinutes": 120,
      "Lifecycle": {
        "MoveToColdStorageAfterDays": 90,
        "DeleteAfterDays": 2555
      },
      "RecoveryPointTags": {
        "Environment": "Production",
        "Compliance": "HIPAA"
      }
    }
  ]
}
```

**Create backup plan:**

```bash
aws backup create-backup-plan \
  --backup-plan file://backup-plan.json

# Assign resources
aws backup create-backup-selection \
  --backup-plan-id <plan-id> \
  --backup-selection '{
    "SelectionName": "HIPAA-Resources",
    "IamRoleArn": "arn:aws:iam::ACCOUNT:role/AWSBackupRole",
    "Resources": [
      "arn:aws:rds:us-east-1:ACCOUNT:db:hipaa-db",
      "arn:aws:ec2:us-east-1:ACCOUNT:volume/vol-xxxxx"
    ]
  }'
```

### Backup Vault Security

**Vault with encryption and access control:**

```bash
aws backup create-backup-vault \
  --backup-vault-name HIPAA-Vault \
  --encryption-key-arn arn:aws:kms:us-east-1:ACCOUNT:key/xxxxx

# Apply vault policy (prevent deletion)
aws backup put-backup-vault-access-policy \
  --backup-vault-name HIPAA-Vault \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Deny",
        "Principal": "*",
        "Action": "backup:DeleteRecoveryPoint",
        "Resource": "*"
      }
    ]
  }'
```

---

## 4. Disaster Recovery Plan

### Multi-AZ vs Multi-Region

**Multi-AZ (Default):**
- ✅ Protects against AZ failures
- ✅ Automatic failover (RDS ~2 min)
- ✅ No additional configuration needed
- ❌ Doesn't protect against region-wide outage

**Multi-Region (Recommended for Critical Systems):**
- ✅ Protects against region failures
- ✅ Compliance with data residency requirements
- ❌ More complex setup
- ❌ Higher cost

### Pilot Light DR Architecture

**Normal State:**
- Primary region: Full production stack
- DR region: Minimal resources (just data backups)

**Failure State:**
- Spin up infrastructure in DR region
- Restore database from snapshot
- Update DNS to DR region

**RTO:** 1-4 hours (manual process)
**RPO:** Last backup (typically < 1 hour with automated snapshots)

### DR Procedure

**1. Database Restore in DR Region:**

```bash
# Copy latest snapshot to DR region (automated)
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier arn:aws:rds:us-east-1:ACCOUNT:snapshot:rds:hipaa-db-2025-11-26-03-00 \
  --target-db-snapshot-identifier hipaa-db-dr-2025-11-26 \
  --region us-west-2 \
  --kms-key-id arn:aws:kms:us-west-2:ACCOUNT:key/xxxxx

# Restore from snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier hipaa-db-dr \
  --db-snapshot-identifier hipaa-db-dr-2025-11-26 \
  --vpc-security-group-ids sg-xxxxx \
  --db-subnet-group-name data-subnet-group-dr \
  --region us-west-2
```

**2. Deploy Application Infrastructure:**

```bash
# Using Infrastructure as Code (Terraform/CloudFormation)
terraform apply -var="region=us-west-2" -var="environment=dr"
```

**3. Update DNS:**

```bash
# Route53 health check detects failure, automatic failover
# OR manual update
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z3AADJGX6KTTL2",
          "DNSName": "dr-alb-123456.us-west-2.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```

**4. Validate:**
- Test application functionality
- Verify database connectivity
- Check monitoring/alerting
- Confirm audit logging

---

## 5. Backup Testing Procedures

### Quarterly Restore Tests (Required)

**2025 HIPAA Update:** Formal testing required at least quarterly.

**Test Plan:**

```markdown
# Q4 2025 Backup Restore Test

**Date:** 2025-11-26
**Tester:** DevOps Team
**Scope:** RDS database restore

## Steps:
1. [ ] Identify latest automated snapshot
2. [ ] Create test RDS instance from snapshot
3. [ ] Validate data integrity (row counts, checksums)
4. [ ] Test application connectivity
5. [ ] Verify encryption status
6. [ ] Measure recovery time
7. [ ] Document results
8. [ ] Delete test instance

## Results:
- Snapshot ID: rds:hipaa-db-2025-11-26-03-00
- Restore time: 12 minutes
- Data integrity: ✅ Verified
- Application connectivity: ✅ Success
- Issues: None

## Next Test: 2026-02-26
```

### Automated Testing (Recommended)

**Lambda function to test restores:**

```python
import boto3
import datetime

def lambda_handler(event, context):
    rds = boto3.client('rds')

    # Get latest automated snapshot
    snapshots = rds.describe_db_snapshots(
        DBInstanceIdentifier='hipaa-db',
        SnapshotType='automated',
        MaxRecords=1
    )

    snapshot_id = snapshots['DBSnapshots'][0]['DBSnapshotIdentifier']

    # Restore to test instance
    test_instance = f"restore-test-{datetime.datetime.now().strftime('%Y%m%d')}"

    rds.restore_db_instance_from_db_snapshot(
        DBInstanceIdentifier=test_instance,
        DBSnapshotIdentifier=snapshot_id,
        VpcSecurityGroupIds=['sg-test'],
        DBSubnetGroupName='test-subnet-group'
    )

    # Schedule cleanup Lambda (delete after validation)
    # ... (omitted for brevity)

    return {'statusCode': 200, 'message': f'Restore test initiated: {test_instance}'}
```

**Schedule:** Weekly automated test, quarterly manual validation.

---

## 6. Data Retention Policy

### HIPAA Retention Requirements

**General Rule:** 6 years from creation or last use.

**Varies by:**
- State law (some require 7+ years)
- Type of record (adult vs minor)
- Specific regulations (Medicare = 10 years)

### Retention by Data Type

| Data Type | Retention | Storage Class | Notes |
|-----------|-----------|---------------|-------|
| Active PHI | Indefinite | Standard | In production database |
| Database backups | 7 years | Glacier Deep Archive | Automated snapshots → manual yearly |
| Audit logs | 7 years | Standard → Glacier | CloudWatch retention |
| CloudTrail logs | 7 years | S3 Standard → Glacier | Immutable |
| Application logs | 2 years | S3 IA → Glacier | Non-PHI logs |
| Deleted patient records | 7 years | Glacier Deep Archive | Hard delete after retention |

### Deletion Procedure

**After retention period:**

1. Identify records for deletion
2. Audit log the deletion request
3. Verify compliance (legal review)
4. Perform hard delete (no soft delete for PHI!)
5. Verify deletion (no recoverable copies)
6. Document deletion in audit trail

---

## Backup & DR Checklist

**RDS Backups:**
- [ ] Automated backups enabled (30-day retention)
- [ ] Backup window during low traffic
- [ ] Point-in-time recovery tested
- [ ] Manual snapshots for long-term (monthly)
- [ ] Cross-region snapshots for DR
- [ ] Deletion protection enabled

**S3 Backups:**
- [ ] Versioning enabled
- [ ] Lifecycle policies configured
- [ ] Cross-region replication (if DR required)
- [ ] All buckets encrypted
- [ ] MFA delete enabled (optional)

**AWS Backup:**
- [ ] Backup vault created with encryption
- [ ] Backup plan configured
- [ ] Resources assigned to plan
- [ ] Vault access policy (deny deletion)
- [ ] Backup compliance reporting

**Disaster Recovery:**
- [ ] DR region identified
- [ ] Cross-region snapshot automation
- [ ] Infrastructure as Code for DR region
- [ ] Route53 health checks configured
- [ ] DR procedure documented
- [ ] DR drill conducted (annually)

**Testing:**
- [ ] Quarterly restore tests scheduled
- [ ] Test results documented
- [ ] RTO/RPO measured and meet requirements
- [ ] Restore procedure documented
- [ ] Automated testing (optional)

**Retention:**
- [ ] Retention policy documented
- [ ] Lifecycle policies enforce retention
- [ ] Deletion procedures defined
- [ ] Legal review of retention periods

---

## Next Steps

1. **Complete DevOps setup:** [Final compliance checklist →](05-checklist.md)
2. **Review monitoring:** [Ensure alerting configured →](03-monitoring.md)

---

## Additional Resources

- [AWS Backup Documentation](https://docs.aws.amazon.com/aws-backup/)
- [RDS Backup and Restore](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_CommonTasks.BackupRestore.html)
- [S3 Versioning and Lifecycle](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html)

---

_Last Updated: November 2025_
_Next: [DevOps Checklist →](05-checklist.md)_
