# Audit Logging

> Recording and monitoring all access to ePHI for HIPAA compliance

---

## Overview

**Audit Controls** are a required technical safeguard under HIPAA Security Rule § 164.312(b). Every access, modification, or deletion of ePHI must be logged with sufficient detail to reconstruct "who did what, when, where, and why."

Audit logs serve two critical purposes:
1. **Breach Detection** - Identify unauthorized access or suspicious activity
2. **Compliance Evidence** - Demonstrate due diligence during audits or investigations

---

## HIPAA Requirements

### Audit Controls (§ 164.312(b))

**Required Implementation:**

"Implement hardware, software, and/or procedural mechanisms that record and examine activity in information systems that contain or use electronic protected health information."

**Key Requirements:**

1. **Record all ePHI access** - Every read, write, delete operation
2. **Sufficient detail** - Who, what, when, where, and result
3. **Immutable logs** - Cannot be altered or deleted by users
4. **Retention** - Minimum 6 years (§ 164.316(b)(2)(i))
5. **Regular review** - Logs must be examined for anomalies

---

## What to Log

### Application Audit Trails

**Required Events for All ePHI Access:**

#### 1. Data Operations
- **Create** - New PHI records created
- **Read** - PHI viewed or accessed
- **Update** - PHI modified
- **Delete** - PHI removed (hard delete only!)

#### 2. Authentication Events
- Login attempts (success and failure)
- Logout events
- MFA verification (success and failure)
- Password changes
- Account lockouts

#### 3. Authorization Events
- Permission grants/revocations
- Role assignments/changes
- Access denied attempts (very important!)
- Privilege escalation

#### 4. System Events
- Configuration changes
- Security policy updates
- Encryption key access
- Backup/restore operations
- Database schema changes

#### 5. Administrative Actions
- User account creation/deletion
- Role modifications
- Policy changes
- Audit log access (meta-logging!)

### System-Level Audit Trails

- Successful/failed logon attempts
- User ID/username
- Date and time of each logon/logoff
- Devices used (IP address, user agent)
- Applications accessed

---

## Log Structure

### Required Fields

Every audit log entry must capture:

```typescript
interface AuditLogEntry {
  // WHO - User identification
  userId: string;              // Unique user ID (UUID)
  username: string;            // Username or email
  userRole: string;            // User's role at time of action

  // WHAT - Action details
  action: AuditAction;         // Enum: PHI_READ, PHI_WRITE, PHI_DELETE, etc.
  resource: string;            // Resource type: 'patient', 'appointment', etc.
  resourceId: string;          // Specific record ID (UUID)

  // WHEN - Timestamp
  timestamp: Date;             // ISO 8601 format, UTC timezone

  // WHERE - Source information
  ipAddress: string;           // Client IP address
  userAgent?: string;          // Browser/client info
  source: string;              // 'web', 'mobile', 'api', 'internal'

  // RESULT - Outcome
  status: 'success' | 'failure';
  errorCode?: string;          // If failed, why?

  // CONTEXT - Additional details (optional but recommended)
  details?: Record<string, any>;  // Additional context (NO PHI!)
  requestId?: string;          // Correlation ID for distributed tracing
}

enum AuditAction {
  // PHI Operations
  PHI_READ = 'PHI_READ',
  PHI_WRITE = 'PHI_WRITE',
  PHI_DELETE = 'PHI_DELETE',
  PHI_EXPORT = 'PHI_EXPORT',

  // Authentication
  USER_LOGIN = 'USER_LOGIN',
  USER_LOGOUT = 'USER_LOGOUT',
  MFA_VERIFY = 'MFA_VERIFY',
  PASSWORD_CHANGE = 'PASSWORD_CHANGE',

  // Authorization
  ACCESS_DENIED = 'ACCESS_DENIED',
  ROLE_ASSIGNED = 'ROLE_ASSIGNED',
  PERMISSION_GRANTED = 'PERMISSION_GRANTED',

  // System
  CONFIG_CHANGE = 'CONFIG_CHANGE',
  ENCRYPTION_KEY_ACCESS = 'ENCRYPTION_KEY_ACCESS',
  BACKUP_CREATED = 'BACKUP_CREATED'
}
```

---

## CRITICAL: Never Log PHI Itself!

### ❌ Wrong - PHI in Logs

```javascript
// HIPAA VIOLATION - Contains actual PHI!
auditLog.create({
  action: 'PHI_READ',
  details: {
    patientName: 'John Doe',
    ssn: '123-45-6789',
    diagnosis: 'Type 2 Diabetes'
  }
});
```

### ✅ Correct - References Only

```javascript
// HIPAA COMPLIANT - References only, no PHI content
auditLog.create({
  action: 'PHI_READ',
  resource: 'patient',
  resourceId: 'uuid-of-patient-record',
  details: {
    fieldsAccessed: ['name', 'ssn', 'diagnosis'],
    recordType: 'medical_record'
  }
});
```

**Key Principle:** Log **WHAT was accessed** (record IDs), not **the content itself** (actual PHI).

---

## NestJS Implementation

### Audit Logging Service

```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from './prisma.service';
import { CloudWatchLogsService } from './cloudwatch.service';

/**
 * Audit logging service for HIPAA compliance
 * Logs all PHI access and security events
 */
@Injectable()
export class AuditLogService {
  constructor(
    private prisma: PrismaService,
    private cloudwatch: CloudWatchLogsService
  ) {}

  /**
   * Create audit log entry
   * Logs to both database and CloudWatch
   */
  async log(entry: Partial<AuditLogEntry>): Promise<void> {
    const auditEntry = {
      id: this.generateId(),
      timestamp: new Date(),
      status: entry.status || 'success',
      ...entry
    };

    // Log to database (for application queries)
    await this.prisma.auditLog.create({
      data: auditEntry
    });

    // Log to CloudWatch (for long-term retention)
    await this.cloudwatch.putLogEvents({
      logGroupName: '/hipaa/audit-logs',
      logStreamName: this.getLogStreamName(),
      logEvents: [{
        timestamp: auditEntry.timestamp.getTime(),
        message: JSON.stringify(auditEntry)
      }]
    });
  }

  /**
   * Log PHI access (most common use case)
   */
  async logPHIAccess(
    userId: string,
    action: AuditAction,
    resource: string,
    resourceId: string,
    request: Request
  ): Promise<void> {
    await this.log({
      userId,
      username: request.user?.username,
      userRole: request.user?.roles?.join(','),
      action,
      resource,
      resourceId,
      ipAddress: this.getClientIp(request),
      userAgent: request.headers['user-agent'],
      source: this.determineSource(request),
      requestId: request.id  // Correlation ID
    });
  }

  /**
   * Log authentication event
   */
  async logAuthEvent(
    userId: string,
    action: AuditAction,
    status: 'success' | 'failure',
    request: Request,
    errorCode?: string
  ): Promise<void> {
    await this.log({
      userId,
      action,
      status,
      ipAddress: this.getClientIp(request),
      userAgent: request.headers['user-agent'],
      errorCode
    });
  }

  /**
   * Extract client IP (considers proxies)
   */
  private getClientIp(request: Request): string {
    return (
      request.headers['x-forwarded-for']?.split(',')[0] ||
      request.headers['x-real-ip'] ||
      request.connection.remoteAddress ||
      'unknown'
    );
  }

  /**
   * Determine request source
   */
  private determineSource(request: Request): string {
    const userAgent = request.headers['user-agent'] || '';

    if (userAgent.includes('Mobile') || userAgent.includes('Android')) {
      return 'mobile';
    }
    if (userAgent.includes('Postman') || userAgent.includes('curl')) {
      return 'api';
    }
    return 'web';
  }

  private generateId(): string {
    return crypto.randomUUID();
  }

  private getLogStreamName(): string {
    // One log stream per day
    return new Date().toISOString().split('T')[0];
  }
}
```

### Audit Logging Interceptor

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap, catchError } from 'rxjs/operators';
import { Reflector } from '@nestjs/core';

/**
 * Interceptor to automatically log PHI access
 * Applied to controllers/routes that handle PHI
 */
@Injectable()
export class AuditLogInterceptor implements NestInterceptor {
  constructor(
    private auditLog: AuditLogService,
    private reflector: Reflector
  ) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const handler = context.getHandler();

    // Get metadata from @AuditLog() decorator
    const auditConfig = this.reflector.get<AuditConfig>(
      'audit',
      handler
    );

    if (!auditConfig) {
      return next.handle();  // No audit logging for this endpoint
    }

    const startTime = Date.now();

    return next.handle().pipe(
      tap(async (response) => {
        // Success - log the access
        await this.auditLog.logPHIAccess(
          request.user?.id,
          auditConfig.action,
          auditConfig.resource,
          this.extractResourceId(request, response, auditConfig),
          request
        );
      }),
      catchError(async (error) => {
        // Failure - log the attempt
        await this.auditLog.log({
          userId: request.user?.id,
          username: request.user?.username,
          action: auditConfig.action,
          resource: auditConfig.resource,
          resourceId: this.extractResourceId(request, null, auditConfig),
          status: 'failure',
          errorCode: error.status || 'UNKNOWN_ERROR',
          ipAddress: this.getClientIp(request),
          timestamp: new Date()
        });

        throw error;  // Re-throw to maintain error flow
      })
    );
  }

  /**
   * Extract resource ID from request params or response
   */
  private extractResourceId(
    request: any,
    response: any,
    config: AuditConfig
  ): string {
    // Try URL params first (e.g., /patients/:id)
    if (request.params[config.idParam || 'id']) {
      return request.params[config.idParam || 'id'];
    }

    // Try response body (for CREATE operations)
    if (response?.id) {
      return response.id;
    }

    return 'unknown';
  }

  private getClientIp(request: Request): string {
    return (
      request.headers['x-forwarded-for']?.split(',')[0] ||
      request.connection.remoteAddress ||
      'unknown'
    );
  }
}

/**
 * Configuration for audit logging
 */
interface AuditConfig {
  action: AuditAction;
  resource: string;
  idParam?: string;  // URL parameter name for resource ID
}
```

### Custom Decorator

```typescript
import { SetMetadata } from '@nestjs/common';

/**
 * Decorator to enable audit logging on endpoint
 * Usage: @AuditLog({ action: AuditAction.PHI_READ, resource: 'patient' })
 */
export const AuditLog = (config: AuditConfig) =>
  SetMetadata('audit', config);
```

### Controller Usage

```typescript
@Controller('patients')
@UseGuards(JwtAuthGuard, RolesGuard)
@UseInterceptors(AuditLogInterceptor)  // Enable audit logging
export class PatientsController {
  constructor(
    private patientsService: PatientsService,
    private auditLog: AuditLogService
  ) {}

  /**
   * Get patient details - audit logged automatically
   */
  @Get(':id')
  @Roles(Role.PROVIDER, Role.NURSE)
  @AuditLog({ action: AuditAction.PHI_READ, resource: 'patient' })
  async getPatient(@Param('id') id: string) {
    return this.patientsService.findOne(id);
  }

  /**
   * Update patient - audit logged automatically
   */
  @Patch(':id')
  @Roles(Role.PROVIDER)
  @AuditLog({ action: AuditAction.PHI_WRITE, resource: 'patient' })
  async updatePatient(
    @Param('id') id: string,
    @Body() updateDto: UpdatePatientDto
  ) {
    return this.patientsService.update(id, updateDto);
  }

  /**
   * Delete patient - audit logged with additional context
   */
  @Delete(':id')
  @Roles(Role.PROVIDER, Role.SUPER_ADMIN)
  @AuditLog({ action: AuditAction.PHI_DELETE, resource: 'patient' })
  async deletePatient(
    @Param('id') id: string,
    @User() user: UserEntity,
    @Req() request: Request
  ) {
    // Manual audit log with additional context
    await this.auditLog.log({
      userId: user.id,
      username: user.username,
      action: AuditAction.PHI_DELETE,
      resource: 'patient',
      resourceId: id,
      ipAddress: request.ip,
      details: {
        reason: 'Patient requested deletion',  // Document why
        approvedBy: user.id
      }
    });

    // Perform hard delete (no soft delete for PHI!)
    return this.patientsService.remove(id);
  }
}
```

---

## Database Schema

### Audit Logs Table

```sql
CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- WHO
  user_id UUID REFERENCES users(id),
  username VARCHAR(255),
  user_role VARCHAR(100),

  -- WHAT
  action VARCHAR(50) NOT NULL,
  resource VARCHAR(100) NOT NULL,
  resource_id VARCHAR(255),

  -- WHEN
  timestamp TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

  -- WHERE
  ip_address VARCHAR(45),  -- IPv6 compatible
  user_agent TEXT,
  source VARCHAR(50),

  -- RESULT
  status VARCHAR(20) NOT NULL DEFAULT 'success',
  error_code VARCHAR(100),

  -- CONTEXT
  details JSONB,
  request_id VARCHAR(100),

  -- Indexes for querying
  CONSTRAINT audit_logs_pkey PRIMARY KEY (id)
);

-- Indexes for common queries
CREATE INDEX idx_audit_logs_user_id ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_resource ON audit_logs(resource, resource_id);
CREATE INDEX idx_audit_logs_timestamp ON audit_logs(timestamp DESC);
CREATE INDEX idx_audit_logs_action ON audit_logs(action);
CREATE INDEX idx_audit_logs_status ON audit_logs(status);

-- GIN index for JSONB details column
CREATE INDEX idx_audit_logs_details ON audit_logs USING GIN (details);
```

### Prisma Schema

```prisma
model AuditLog {
  id         String   @id @default(uuid())

  // WHO
  userId     String?  @map("user_id")
  username   String?
  userRole   String?  @map("user_role")

  // WHAT
  action     String
  resource   String
  resourceId String?  @map("resource_id")

  // WHEN
  timestamp  DateTime @default(now())

  // WHERE
  ipAddress  String?  @map("ip_address")
  userAgent  String?  @map("user_agent")
  source     String?

  // RESULT
  status     String   @default("success")
  errorCode  String?  @map("error_code")

  // CONTEXT
  details    Json?
  requestId  String?  @map("request_id")

  // Relations
  user       User?    @relation(fields: [userId], references: [id])

  @@index([userId])
  @@index([resource, resourceId])
  @@index([timestamp(sort: Desc)])
  @@index([action])
  @@map("audit_logs")
}
```

---

## AWS CloudWatch Logs

### Log Group Configuration

```typescript
/**
 * CloudWatch Log Group for audit logs
 * Required: 6+ year retention for HIPAA
 */
const auditLogGroup = new logs.LogGroup(stack, 'AuditLogGroup', {
  logGroupName: '/hipaa/audit-logs',

  // HIPAA REQUIRED: 6-year retention minimum
  retention: logs.RetentionDays.SIX_YEARS,  // 2192 days

  // Encryption at rest
  encryptionKey: kmsKey,

  // Prevent accidental deletion
  removalPolicy: cdk.RemovalPolicy.RETAIN
});
```

### Structured Logging with Winston

```typescript
import * as winston from 'winston';
import * as CloudWatchTransport from 'winston-cloudwatch';

/**
 * Configure Winston for structured logging to CloudWatch
 */
const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()  // Structured JSON logs
  ),
  transports: [
    // Console (for development)
    new winston.transports.Console({
      format: winston.format.simple()
    }),

    // CloudWatch Logs (for production)
    new CloudWatchTransport({
      logGroupName: '/hipaa/audit-logs',
      logStreamName: () => {
        // One stream per day
        return new Date().toISOString().split('T')[0];
      },
      awsRegion: process.env.AWS_REGION,
      jsonMessage: true,  // Send as JSON
      messageFormatter: (logObject) => {
        // Ensure PHI is not logged
        return JSON.stringify(logObject);
      }
    })
  ]
});

/**
 * Log audit event to CloudWatch
 */
logger.info('PHI Access', {
  audit: {
    userId: 'user-uuid',
    action: 'PHI_READ',
    resource: 'patient',
    resourceId: 'patient-uuid',
    timestamp: new Date().toISOString()
  }
});
```

---

## Log Retention & Immutability

### HIPAA Requirement: 6-Year Retention

**Regulation:** § 164.316(b)(2)(i)

"Retain documentation... for 6 years from the date of its creation or the date when it last was in effect, whichever is later."

### Implementation Strategy

**Hot Storage (Recent Logs):**
- Database: Last 30-90 days for fast application queries
- CloudWatch Logs: Last 30 days for debugging

**Cold Storage (Historical Logs):**
- S3 with Glacier transition: 90 days to 6+ years
- Immutable with S3 Object Lock

```typescript
/**
 * S3 bucket for long-term audit log retention
 * Immutable storage with Object Lock
 */
const auditLogBucket = new s3.Bucket(stack, 'AuditLogArchive', {
  bucketName: 'hipaa-audit-logs-archive',

  // HIPAA: Encryption at rest
  encryption: s3.BucketEncryption.KMS,
  encryptionKey: kmsKey,

  // Block all public access
  blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,

  // Versioning (protect against deletion)
  versioned: true,

  // IMMUTABLE: Object Lock prevents deletion/modification
  objectLockEnabled: true,

  // Lifecycle: Move to Glacier after 90 days
  lifecycleRules: [
    {
      id: 'ArchiveAuditLogs',
      enabled: true,

      // Transition to Glacier after 90 days
      transitions: [
        {
          storageClass: s3.StorageClass.GLACIER,
          transitionAfter: cdk.Duration.days(90)
        }
      ],

      // Delete after 7 years (6 years + 1 year buffer)
      expiration: cdk.Duration.days(2555)
    }
  ]
});

// Object Lock configuration (WORM - Write Once, Read Many)
const cfnBucket = auditLogBucket.node.defaultChild as s3.CfnBucket;
cfnBucket.objectLockConfiguration = {
  objectLockEnabled: 'Enabled',
  rule: {
    defaultRetention: {
      mode: 'GOVERNANCE',  // Can be overridden by admin
      years: 6
    }
  }
};
```

### CloudWatch to S3 Export

```typescript
/**
 * Lambda function to export CloudWatch logs to S3
 * Runs daily to archive logs
 */
const exportLambda = new lambda.Function(stack, 'ExportAuditLogs', {
  runtime: lambda.Runtime.NODEJS_18_X,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda/export-logs'),
  environment: {
    LOG_GROUP_NAME: '/hipaa/audit-logs',
    BUCKET_NAME: auditLogBucket.bucketName
  },
  timeout: cdk.Duration.minutes(15)
});

// Schedule daily export
const rule = new events.Rule(stack, 'ExportSchedule', {
  schedule: events.Schedule.cron({ hour: '2', minute: '0' })  // 2 AM daily
});
rule.addTarget(new targets.LambdaFunction(exportLambda));

// Grant permissions
auditLogGroup.grantRead(exportLambda);
auditLogBucket.grantWrite(exportLambda);
```

---

## Querying Audit Logs

### CloudWatch Insights Queries

**Find all PHI access by user:**

```sql
fields @timestamp, action, resource, resourceId, status
| filter userId = "user-uuid-here"
| filter action = "PHI_READ" or action = "PHI_WRITE" or action = "PHI_DELETE"
| sort @timestamp desc
| limit 100
```

**Detect failed access attempts:**

```sql
fields @timestamp, userId, username, action, resource, errorCode
| filter status = "failure"
| stats count() by userId, action
| sort count desc
```

**Unusual activity (access outside business hours):**

```sql
fields @timestamp, userId, username, action, resource, resourceId
| filter @timestamp >= ago(7d)
| filter datefloor(@timestamp, 1h) < 8 or datefloor(@timestamp, 1h) > 18
| stats count() by userId, username
```

### Application Queries (Database)

```typescript
/**
 * Get audit trail for specific patient
 */
async getPatientAuditTrail(patientId: string): Promise<AuditLog[]> {
  return this.prisma.auditLog.findMany({
    where: {
      resource: 'patient',
      resourceId: patientId
    },
    orderBy: {
      timestamp: 'desc'
    },
    include: {
      user: {
        select: {
          username: true,
          email: true
        }
      }
    }
  });
}

/**
 * Find suspicious activity (multiple failed access attempts)
 */
async findSuspiciousActivity(hours: number = 24): Promise<any[]> {
  const since = new Date(Date.now() - hours * 60 * 60 * 1000);

  return this.prisma.$queryRaw`
    SELECT user_id, username, COUNT(*) as failed_attempts
    FROM audit_logs
    WHERE status = 'failure'
      AND action = 'ACCESS_DENIED'
      AND timestamp > ${since}
    GROUP BY user_id, username
    HAVING COUNT(*) > 5
    ORDER BY failed_attempts DESC
  `;
}
```

---

## Alerting & Monitoring

### CloudWatch Alarms

```typescript
/**
 * Alarm: High rate of failed PHI access attempts
 */
const failedAccessMetric = new logs.MetricFilter(stack, 'FailedAccess', {
  logGroup: auditLogGroup,
  filterPattern: logs.FilterPattern.all(
    logs.FilterPattern.stringValue('$.status', '=', 'failure'),
    logs.FilterPattern.stringValue('$.action', '=', 'ACCESS_DENIED')
  ),
  metricNamespace: 'HIPAA/Security',
  metricName: 'FailedPHIAccess',
  metricValue: '1'
});

const alarm = new cloudwatch.Alarm(stack, 'FailedAccessAlarm', {
  metric: failedAccessMetric.metric({
    statistic: 'Sum',
    period: cdk.Duration.minutes(5)
  }),
  threshold: 10,  // More than 10 failures in 5 minutes
  evaluationPeriods: 1,
  alarmDescription: 'High rate of failed PHI access attempts',
  actionsEnabled: true
});

// SNS notification
const topic = new sns.Topic(stack, 'SecurityAlerts');
alarm.addAlarmAction(new actions.SnsAction(topic));
```

**Alerts to implement:**
- ✅ Failed authentication attempts (brute force detection)
- ✅ Access denied events (permission issues)
- ✅ PHI deletion events (rare, should be reviewed)
- ✅ After-hours access (unusual activity)
- ✅ Bulk PHI exports (potential breach)
- ✅ Multiple users from same IP (account sharing)

---

## Common Mistakes & How to Avoid Them

### ❌ Mistake 1: Logging PHI Content
**Problem:** Audit logs contain actual PHI (names, diagnoses, SSN)
**HIPAA Violation:** Logs are not encrypted/protected like primary PHI storage
**Solution:** Only log record IDs and metadata, never PHI content

### ❌ Mistake 2: Mutable Logs
**Problem:** Application can modify or delete audit logs
**HIPAA Violation:** § 164.312(b) requires immutable audit trails
**Solution:** Write-only permissions for application, use S3 Object Lock

### ❌ Mistake 3: Insufficient Retention
**Problem:** Logs deleted after 90 days or 1 year
**HIPAA Violation:** § 164.316(b)(2)(i) requires 6-year retention
**Solution:** CloudWatch 6-year retention + S3 Glacier archival

### ❌ Mistake 4: Not Logging Failed Attempts
**Problem:** Only successful PHI access is logged
**HIPAA Risk:** Miss potential security incidents
**Solution:** Log both success AND failure, especially access denials

### ❌ Mistake 5: No Log Monitoring
**Problem:** Logs generated but never reviewed
**HIPAA Risk:** Breaches go undetected for months
**Solution:** Automated alerts + weekly manual review

### ❌ Mistake 6: Generic Logging Only
**Problem:** Only application error logs, no specific PHI access logs
**HIPAA Violation:** Cannot demonstrate who accessed what PHI
**Solution:** Dedicated audit log service for all PHI operations

---

## Testing Audit Logging

### Unit Tests

```typescript
describe('AuditLogService', () => {
  it('should log PHI access', async () => {
    await auditLogService.logPHIAccess(
      'user-id',
      AuditAction.PHI_READ,
      'patient',
      'patient-id',
      mockRequest
    );

    const logs = await prisma.auditLog.findMany({
      where: { userId: 'user-id' }
    });

    expect(logs).toHaveLength(1);
    expect(logs[0].action).toBe('PHI_READ');
  });

  it('should not log PHI content', async () => {
    await auditLogService.log({
      action: AuditAction.PHI_READ,
      details: {
        patientName: 'John Doe'  // This should fail validation
      }
    });

    // Should throw or sanitize
  });
});
```

### Integration Tests

```typescript
describe('Audit Logging E2E', () => {
  it('should log when patient is accessed', async () => {
    const response = await request(app.getHttpServer())
      .get('/patients/test-patient-id')
      .set('Authorization', `Bearer ${userToken}`)
      .expect(200);

    // Verify audit log was created
    const logs = await prisma.auditLog.findMany({
      where: {
        resource: 'patient',
        resourceId: 'test-patient-id'
      }
    });

    expect(logs).toHaveLength(1);
    expect(logs[0].action).toBe('PHI_READ');
  });

  it('should log failed access attempts', async () => {
    const response = await request(app.getHttpServer())
      .get('/patients/unauthorized-patient')
      .set('Authorization', `Bearer ${userToken}`)
      .expect(403);

    // Verify failed attempt was logged
    const logs = await prisma.auditLog.findMany({
      where: {
        status: 'failure',
        action: 'ACCESS_DENIED'
      }
    });

    expect(logs.length).toBeGreaterThan(0);
  });
});
```

---

## Checklist

Before deploying audit logging:

- [ ] **All PHI operations logged** - Create, Read, Update, Delete
- [ ] **Authentication events logged** - Login, logout, MFA
- [ ] **Authorization failures logged** - Access denied events
- [ ] **No PHI in audit logs** - Only IDs and metadata
- [ ] **Structured logging** - JSON format with required fields
- [ ] **Database audit logs** - Last 30-90 days for queries
- [ ] **CloudWatch Logs configured** - 6-year retention
- [ ] **S3 archival setup** - Glacier transition after 90 days
- [ ] **S3 Object Lock enabled** - Immutable storage (WORM)
- [ ] **Audit log encryption** - KMS encryption at rest
- [ ] **Failed attempts logged** - Track security incidents
- [ ] **CloudWatch alarms** - Alert on suspicious activity
- [ ] **Regular log reviews** - Weekly manual examination
- [ ] **Incident response plan** - What to do when alarms trigger

---

## Additional Resources

**HIPAA Guidance:**
- [HIPAA Audit Log Requirements](https://www.kiteworks.com/hipaa-compliance/hipaa-audit-log-requirements/)
- [HIPAA Retention Requirements](https://www.hipaajournal.com/hipaa-retention-requirements/)
- [Audit Trail Requirements](https://compliancy-group.com/hipaa-audit-log-requirements/)

**AWS Documentation:**
- [CloudWatch Logs User Guide](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/)
- [S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)

**Implementation:**
- [Winston Logger](https://github.com/winstonjs/winston)
- [NestJS Logging](https://docs.nestjs.com/techniques/logger)

---

## Next Steps

Continue to the next section:

**[→ Data Handling](04-data-handling.md)**

Or explore related topics:
- [Authentication](01-authentication.md) - Log auth events
- [Backend Checklist](05-checklist.md) - Pre-deployment verification

---

*Last Updated: November 2025*
*Part of the HIPAA Compliance Handbook - Backend Playbook*
