---
layout: default
title: Audit Logging
parent: Backend Playbook
nav_order: 4
---

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
  userId: string; // Unique user ID (UUID)
  username: string; // Username or email
  userRole: string; // User's role at time of action

  // WHAT - Action details
  action: AuditAction; // Enum: PHI_READ, PHI_WRITE, PHI_DELETE, etc.
  resource: string; // Resource type: 'patient', 'appointment', etc.
  resourceId: string; // Specific record ID (UUID)

  // WHEN - Timestamp
  timestamp: Date; // ISO 8601 format, UTC timezone

  // WHERE - Source information
  ipAddress: string; // Client IP address
  userAgent?: string; // Browser/client info
  source: string; // 'web', 'mobile', 'api', 'internal'

  // RESULT - Outcome
  status: "success" | "failure";
  errorCode?: string; // If failed, why?

  // CONTEXT - Additional details (optional but recommended)
  details?: Record<string, any>; // Additional context (NO PHI!)
  requestId?: string; // Correlation ID for distributed tracing
}

enum AuditAction {
  // PHI Operations
  PHI_READ = "PHI_READ",
  PHI_WRITE = "PHI_WRITE",
  PHI_DELETE = "PHI_DELETE",
  PHI_EXPORT = "PHI_EXPORT",

  // Authentication
  USER_LOGIN = "USER_LOGIN",
  USER_LOGOUT = "USER_LOGOUT",
  MFA_VERIFY = "MFA_VERIFY",
  PASSWORD_CHANGE = "PASSWORD_CHANGE",

  // Authorization
  ACCESS_DENIED = "ACCESS_DENIED",
  ROLE_ASSIGNED = "ROLE_ASSIGNED",
  PERMISSION_GRANTED = "PERMISSION_GRANTED",

  // System
  CONFIG_CHANGE = "CONFIG_CHANGE",
  ENCRYPTION_KEY_ACCESS = "ENCRYPTION_KEY_ACCESS",
  BACKUP_CREATED = "BACKUP_CREATED",
}
```

---

## CRITICAL: Never Log PHI Itself!

### ❌ Wrong - PHI in Logs

```javascript
// HIPAA VIOLATION - Contains actual PHI!
auditLog.create({
  action: "PHI_READ",
  details: {
    patientName: "John Doe",
    ssn: "123-45-6789",
    diagnosis: "Type 2 Diabetes",
  },
});
```

### ✅ Correct - References Only

```javascript
// HIPAA COMPLIANT - References only, no PHI content
auditLog.create({
  action: "PHI_READ",
  resource: "patient",
  resourceId: "uuid-of-patient-record",
  details: {
    fieldsAccessed: ["name", "ssn", "diagnosis"],
    recordType: "medical_record",
  },
});
```

**Key Principle:** Log **WHAT was accessed** (record IDs), not **the content itself** (actual PHI).

---

## NestJS Implementation

### Audit Logging Service

```typescript
// Create audit log service
@Injectable()
export class AuditLogService {
  async log(entry: AuditLogEntry) {
    // 1. Log to database (recent queries)
    await prisma.auditLog.create({ data: entry })

    // 2. Log to external system (long-term retention)
    await externalLogs.send(JSON.stringify(entry))
  }

  async logPHIAccess(userId, action, resourceId, request) {
    await this.log({
      userId, action, resourceId,
      ipAddress: this.getClientIp(request),
      timestamp: new Date()
    })
  }
}
```

**Key points:**
- Log to TWO places: database (fast queries) + external (long-term)
- Capture WHO, WHAT, WHEN, WHERE, RESULT
- Extract client IP (handle proxies)
- Never log PHI content, only IDs

### Audit Logging Interceptor

```typescript
// Interceptor to automatically log PHI access
@Injectable()
export class AuditLogInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler) {
    const request = context.switchToHttp().getRequest()
    const config = this.reflector.get("audit", context.getHandler())

    return next.handle().pipe(
      tap(async () => {
        // ✅ Success: log access
        await this.auditLog.logPHIAccess(...)
      }),
      catchError(async (error) => {
        // ✅ Failure: log attempt
        await this.auditLog.log({ status: "failure", ... })
        throw error
      })
    )
  }
}
```

### Controller Usage

```typescript
@Controller("patients")
@UseInterceptors(AuditLogInterceptor)
export class PatientsController {
  @Get(":id")
  @AuditLog({ action: "PHI_READ", resource: "patient" })
  async getPatient(@Param("id") id: string) {
    return this.patientsService.findOne(id)
  }

  @Delete(":id")
  @AuditLog({ action: "PHI_DELETE", resource: "patient" })
  async deletePatient(@Param("id") id: string) {
    // Log with context before deletion
    await this.auditLog.log({
      action: "PHI_DELETE",
      resourceId: id,
      details: { reason: "Retention period expired" }
    })

    // Hard delete only!
    return this.patientsService.remove(id)
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

## External Log Storage

**DevOps Checklist:**

- ✅ External log service configured (CloudWatch, Datadog, etc.)
- ✅ 6-year retention minimum
- ✅ Logs encrypted at rest
- ✅ Logs immutable (write-only access for app)
- ✅ Log service protected from accidental deletion

**What to verify:**
- Logs sent to external service in structured JSON format
- Retention policy set correctly
- Application cannot modify/delete logs
- Encryption enabled on log storage

### Structured Logging Example

```typescript
// Configure structured logging
const logger = winston.createLogger({
  format: winston.format.json(),
  transports: [
    new ExternalLogTransport({  // CloudWatch, Datadog, etc.
      logGroup: "/hipaa/audit-logs",
      retention: "6 years"
    })
  ]
})

// Log audit event
logger.info("PHI Access", {
  userId: "uuid", action: "PHI_READ",
  resourceId: "patient-uuid"
})
```

---

## Log Retention & Immutability

### HIPAA Requirement: 6-Year Retention

**Regulation:** § 164.316(b)(2)(i)

"Retain documentation... for 6 years from the date of its creation or the date when it last was in effect, whichever is later."

### Implementation Strategy

**Hot Storage (Recent Logs):**
- Database: Last 30-90 days for fast queries
- External logs: Last 30 days for debugging

**Cold Storage (Historical Logs):**
- Object storage (S3/similar): 90 days to 6+ years
- Immutable with write-once-read-many (WORM) protection

**DevOps Checklist:**

- ✅ Object storage bucket configured with encryption
- ✅ Versioning enabled (protect against deletion)
- ✅ Object lock enabled (immutability)
- ✅ Lifecycle rules: transition to cold storage after 90 days
- ✅ Retention: 6-7 years before expiration
- ✅ Daily export job from external logs to object storage

**What to verify:**
- Logs cannot be modified after creation
- Archive transition working automatically
- Export job runs successfully daily
- Old logs accessible when needed

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

**DevOps Checklist:**

- ✅ Alerts configured for failed authentication (>5 in 5 min)
- ✅ Alerts for access denied events
- ✅ Alerts for PHI deletion (rare, review all)
- ✅ Alerts for after-hours access (unusual patterns)
- ✅ Alerts for bulk exports (potential breach)
- ✅ Notifications sent to security team

**Alert Examples:**

```typescript
// ✅ High rate of failed access attempts
if (failedAccessCount > 10 in 5 minutes) {
  sendAlert("Possible brute force attack")
}

// ✅ PHI deletion event
if (action === "PHI_DELETE") {
  sendAlert("PHI deletion occurred", { userId, resourceId })
}

// ✅ After-hours access
if (hour < 8 || hour > 18) {
  logWarning("After-hours PHI access", { userId })
}
```

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

```typescript
// ✅ Test: PHI access is logged
await request.get("/patients/123").expect(200)
const logs = await prisma.auditLog.findMany({ where: { resourceId: "123" } })
expect(logs[0].action).toBe("PHI_READ")

// ✅ Test: Failed access is logged
await request.get("/patients/unauthorized").expect(403)
const logs = await prisma.auditLog.findMany({ where: { status: "failure" } })
expect(logs.length).toBeGreaterThan(0)

// ✅ Test: No PHI in logs
const log = await prisma.auditLog.findFirst()
expect(log.details).not.toContain("patient name")  // PHI prohibited!
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

_Last Updated: November 2025_
_Part of the HIPAA Compliance Handbook - Backend Playbook_
