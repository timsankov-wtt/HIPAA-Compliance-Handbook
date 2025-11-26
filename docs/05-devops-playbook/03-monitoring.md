---
layout: default
title: Monitoring & Logging
parent: DevOps Playbook
nav_order: 3
---

# Monitoring & Logging for HIPAA Compliance

> CloudWatch, CloudTrail, GuardDuty, and Config for audit trails and security monitoring

---

## Logging Requirements

HIPAA requires comprehensive audit trails for all PHI access and system events.

**Must Log:**
- All PHI access (read, write, delete)
- Authentication attempts (success and failures)
- Authorization failures
- Security events (firewall changes, IAM modifications)
- Configuration changes
- API calls

**Retention:** Minimum 6 years (HIPAA recommendation)

---

## 1. CloudWatch Logs

### Purpose
Centralized logging for application logs, system logs, and service logs.

### Log Groups Organization

**Structure:**
```
/aws/ecs/hipaa-app-prod           # ECS application logs
/aws/rds/instance/hipaa-db/postgresql  # RDS database logs
/aws/lambda/audit-processor        # Lambda function logs
/aws/apigateway/api-prod          # API Gateway logs
/aws/vpc/flowlogs                 # Network traffic logs
```

### Configuration

**Create Log Group with Encryption:**

```bash
aws logs create-log-group \
  --log-group-name /aws/ecs/hipaa-app-prod \
  --kms-key-id arn:aws:kms:us-east-1:ACCOUNT:key/xxxxx

# Set retention (6 years = 2190 days)
aws logs put-retention-policy \
  --log-group-name /aws/ecs/hipaa-app-prod \
  --retention-in-days 2190
```

**Retention Options:**
- Audit logs: 2190+ days (6 years)
- Application logs: 365-730 days (1-2 years)
- Debug logs: 30-90 days

### Metric Filters

**Monitor specific events:**

```bash
# Alert on PHI access
aws logs put-metric-filter \
  --log-group-name /aws/ecs/hipaa-app-prod \
  --filter-name PHIAccessCount \
  --filter-pattern '[time, request_id, event_type="PHI_ACCESS", ...]' \
  --metric-transformations \
    metricName=PHIAccessCount,\
    metricNamespace=HIPAA/Audit,\
    metricValue=1

# Alert on failed authentications
aws logs put-metric-filter \
  --log-group-name /aws/ecs/hipaa-app-prod \
  --filter-name FailedAuthCount \
  --filter-pattern '[time, request_id, event="AUTH_FAILED", ...]' \
  --metric-transformations \
    metricName=FailedAuthCount,\
    metricNamespace=HIPAA/Security,\
    metricValue=1
```

### CloudWatch Insights Queries

**Example queries for auditing:**

**All PHI access by user:**
```
fields @timestamp, userId, action, resourceType, resourceId
| filter eventType = "PHI_ACCESS"
| sort @timestamp desc
| limit 1000
```

**Failed authentication attempts:**
```
fields @timestamp, userId, ipAddress, reason
| filter event = "AUTH_FAILED"
| stats count() by userId
| sort count desc
```

**Unusual access patterns:**
```
fields @timestamp, userId, resourceId
| filter eventType = "PHI_ACCESS"
| stats count() by userId, bin(5m)
| filter count > 100  # More than 100 accesses in 5 min
```

---

## 2. CloudTrail

### Purpose
Audit trail of all AWS API calls (management and data events).

**Critical for HIPAA:** CloudTrail provides immutable logs of who did what, when, and from where.

### Multi-Region Trail Setup

```bash
# Create trail (logs to S3, encrypted)
aws cloudtrail create-trail \
  --name hipaa-audit-trail \
  --s3-bucket-name hipaa-cloudtrail-logs \
  --is-multi-region-trail \
  --enable-log-file-validation \
  --kms-key-id arn:aws:kms:us-east-1:ACCOUNT:key/xxxxx

# Start logging
aws cloudtrail start-logging --name hipaa-audit-trail

# Enable CloudWatch Logs integration
aws cloudtrail update-trail \
  --name hipaa-audit-trail \
  --cloud-watch-logs-log-group-arn arn:aws:logs:us-east-1:ACCOUNT:log-group:/aws/cloudtrail \
  --cloud-watch-logs-role-arn arn:aws:iam::ACCOUNT:role/CloudTrailCloudWatchRole
```

### Log File Validation

**Purpose:** Detect tampering with CloudTrail logs.

**Enable:**
```bash
--enable-log-file-validation
```

**Validate:**
```bash
aws cloudtrail validate-logs \
  --trail-arn arn:aws:cloudtrail:us-east-1:ACCOUNT:trail/hipaa-audit-trail \
  --start-time 2025-11-01T00:00:00Z \
  --end-time 2025-11-26T23:59:59Z
```

### Data Events (Optional)

**Track S3 object access and Lambda invocations:**

```bash
aws cloudtrail put-event-selectors \
  --trail-name hipaa-audit-trail \
  --event-selectors '[{
    "ReadWriteType": "All",
    "IncludeManagementEvents": true,
    "DataResources": [
      {
        "Type": "AWS::S3::Object",
        "Values": ["arn:aws:s3:::hipaa-phi-bucket/*"]
      }
    ]
  }]'
```

**Cost Warning:** Data events are expensive. Use only for buckets containing PHI.

### CloudTrail S3 Bucket Policy

**Prevent deletion and public access:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSCloudTrailAclCheck",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::hipaa-cloudtrail-logs"
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::hipaa-cloudtrail-logs/*",
      "Condition": {
        "StringEquals": {"s3:x-amz-acl": "bucket-owner-full-control"}
      }
    },
    {
      "Sid": "DenyDelete",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:DeleteObject",
      "Resource": "arn:aws:s3:::hipaa-cloudtrail-logs/*"
    }
  ]
}
```

---

## 3. VPC Flow Logs

### Purpose
Monitor network traffic for security analysis and compliance.

**Use Cases:**
- Detect unauthorized access attempts
- Identify unusual traffic patterns
- Forensics after security incidents

### Configuration

```bash
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-xxxxx \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/flowlogs \
  --deliver-logs-permission-arn arn:aws:iam::ACCOUNT:role/VPCFlowLogsRole
```

### Query Flow Logs

**Rejected connections (potential attacks):**
```
fields @timestamp, srcAddr, dstAddr, dstPort, action
| filter action = "REJECT"
| stats count() by srcAddr, dstPort
| sort count desc
```

**Traffic to RDS from unexpected sources:**
```
fields @timestamp, srcAddr, dstAddr, dstPort
| filter dstPort = 5432 and dstAddr like /^10\.0\.21\./
| filter srcAddr not like /^10\.0\.11\./  # Not from app subnet
```

---

## 4. AWS GuardDuty

### Purpose
Intelligent threat detection using machine learning.

**Detects:**
- Compromised instances (bitcoin mining, command & control)
- Reconnaissance (port scanning, brute force)
- Unusual API calls (from Tor nodes, unusual locations)
- Data exfiltration attempts

### Enable GuardDuty

```bash
aws guardduty create-detector \
  --enable \
  --finding-publishing-frequency FIFTEEN_MINUTES
```

**Enable in all regions** (threats can originate anywhere).

### Export Findings to S3

```bash
aws guardduty create-publishing-destination \
  --detector-id <detector-id> \
  --destination-type S3 \
  --destination-properties '{
    "DestinationArn": "arn:aws:s3:::guardduty-findings",
    "KmsKeyArn": "arn:aws:kms:us-east-1:ACCOUNT:key/xxxxx"
  }'
```

### Alerting

**SNS Topic for High-Severity Findings:**

```bash
# Create EventBridge rule
aws events put-rule \
  --name guardduty-high-severity \
  --event-pattern '{
    "source": ["aws.guardduty"],
    "detail-type": ["GuardDuty Finding"],
    "detail": {
      "severity": [7, 8, 9]
    }
  }'

# Add SNS target
aws events put-targets \
  --rule guardduty-high-severity \
  --targets "Id=1,Arn=arn:aws:sns:us-east-1:ACCOUNT:security-alerts"
```

---

## 5. AWS Config

### Purpose
Track resource configuration changes and compliance status.

**Benefits:**
- Automated compliance checks (HIPAA rules)
- Configuration history (who changed what)
- Drift detection
- Remediation automation

### Enable AWS Config

```bash
aws configservice put-configuration-recorder \
  --configuration-recorder name=default,roleARN=arn:aws:iam::ACCOUNT:role/ConfigRole \
  --recording-group allSupported=true,includeGlobalResourceTypes=true

aws configservice put-delivery-channel \
  --delivery-channel name=default,s3BucketName=config-bucket,configSnapshotDeliveryProperties={deliveryFrequency=Six_Hours}

aws configservice start-configuration-recorder \
  --configuration-recorder-name default
```

### HIPAA Config Rules

**Enable managed rules:**

```bash
# Require RDS encryption
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "rds-storage-encrypted",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "RDS_STORAGE_ENCRYPTED"
    }
  }'

# Require S3 bucket encryption
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "s3-bucket-server-side-encryption-enabled",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED"
    }
  }'

# Require CloudTrail enabled
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "cloud-trail-enabled",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "CLOUD_TRAIL_ENABLED"
    }
  }'

# Require MFA on root account
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "root-account-mfa-enabled",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "ROOT_ACCOUNT_MFA_ENABLED"
    }
  }'
```

**Additional rules:**
- `s3-bucket-public-read-prohibited`
- `s3-bucket-public-write-prohibited`
- `rds-instance-public-access-check`
- `vpc-flow-logs-enabled`
- `iam-password-policy`
- `access-keys-rotated`

### Conformance Packs (Recommended)

**HIPAA Conformance Pack:**

```bash
aws configservice put-conformance-pack \
  --conformance-pack-name hipaa-security-pack \
  --template-s3-uri s3://aws-configservice-conformance-packs/Operational-Best-Practices-for-HIPAA-Security.yaml
```

**Includes 40+ pre-configured rules** for HIPAA compliance.

---

## 6. Alerting & Incident Response

### CloudWatch Alarms

**Failed authentication rate:**

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name high-failed-auth-rate \
  --alarm-description "Alert on high failed authentication rate" \
  --metric-name FailedAuthCount \
  --namespace HIPAA/Security \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 50 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:ACCOUNT:security-alerts
```

**RDS CPU high:**

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name rds-high-cpu \
  --metric-name CPUUtilization \
  --namespace AWS/RDS \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=DBInstanceIdentifier,Value=hipaa-db \
  --alarm-actions arn:aws:sns:us-east-1:ACCOUNT:devops-alerts
```

### SNS Topics for Notifications

**Create separate topics:**
- `security-alerts` - High priority (GuardDuty, failed auth)
- `compliance-alerts` - Config rule violations
- `devops-alerts` - Operational issues (CPU, disk)

**Integration:**
- Email notifications
- Slack webhooks
- PagerDuty integration
- Lambda for automated response

---

## 7. Log Analysis & Auditing

### Common Audit Queries

**1. All actions by specific user:**

```sql
-- CloudTrail Insights
SELECT eventTime, eventName, sourceIPAddress, userAgent
FROM cloudtrail_logs
WHERE userIdentity.principalId = 'AIDAI...'
ORDER BY eventTime DESC
LIMIT 1000
```

**2. Changes to security groups:**

```
fields @timestamp, userIdentity.principalId, requestParameters.groupId
| filter eventName like /AuthorizeSecurityGroup/
| sort @timestamp desc
```

**3. S3 bucket policy changes:**

```
fields @timestamp, userIdentity.principalId, requestParameters.bucketName
| filter eventName = "PutBucketPolicy"
| display @timestamp, userIdentity.principalId, requestParameters.bucketPolicy
```

**4. RDS snapshot creation/deletion:**

```
fields @timestamp, userIdentity.principalId, eventName, requestParameters.dBSnapshotIdentifier
| filter eventSource = "rds.amazonaws.com" and (eventName = "CreateDBSnapshot" or eventName = "DeleteDBSnapshot")
| sort @timestamp desc
```

### Athena Queries on CloudTrail

**Setup:**

1. Create Athena table from CloudTrail S3 bucket
2. Query using SQL

**Example - Detect root account usage:**

```sql
SELECT eventtime, eventname, sourceipaddress, useragent
FROM cloudtrail_logs
WHERE useridentity.type = 'Root'
  AND eventtime > '2025-11-01'
ORDER BY eventtime DESC;
```

---

## 8. Compliance Dashboards

### CloudWatch Dashboard

**Create custom dashboard:**

```bash
aws cloudwatch put-dashboard \
  --dashboard-name HIPAA-Compliance \
  --dashboard-body file://dashboard-config.json
```

**Widgets to include:**
- Failed authentication count (last 24h)
- PHI access count per hour
- GuardDuty high-severity findings
- Config non-compliant resources
- RDS CPU/storage metrics
- ECS task health

---

## Monitoring Checklist

**CloudWatch Logs:**
- [ ] Log groups created for all services
- [ ] Retention set to 6+ years (audit logs)
- [ ] Encryption enabled (KMS)
- [ ] Metric filters configured
- [ ] Insights queries saved

**CloudTrail:**
- [ ] Multi-region trail enabled
- [ ] Log file validation enabled
- [ ] S3 bucket encrypted and access restricted
- [ ] CloudWatch Logs integration enabled
- [ ] Data events enabled for PHI buckets

**VPC Flow Logs:**
- [ ] Enabled for all VPCs
- [ ] Logs sent to CloudWatch
- [ ] Retention configured

**GuardDuty:**
- [ ] Enabled in all regions
- [ ] Findings exported to S3
- [ ] High-severity alerts configured
- [ ] Reviewed weekly

**AWS Config:**
- [ ] Enabled in all regions
- [ ] HIPAA conformance pack deployed
- [ ] Non-compliance alerts configured
- [ ] Config history retained

**Alerting:**
- [ ] SNS topics created
- [ ] CloudWatch alarms configured
- [ ] GuardDuty → SNS integration
- [ ] Config → SNS integration
- [ ] Incident response procedures documented

---

## Next Steps

1. **Configure backups:** [Backup and DR procedures →](04-backup-dr.md)
2. **Final checklist:** [Complete DevOps checklist →](05-checklist.md)

---

## Additional Resources

- [CloudWatch Logs Insights Query Syntax](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html)
- [CloudTrail Log Event Reference](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-event-reference.html)
- [AWS Config Managed Rules](https://docs.aws.amazon.com/config/latest/developerguide/managed-rules-by-aws-config.html)

---

_Last Updated: November 2025_
_Next: [Backup & Disaster Recovery →](04-backup-dr.md)_
