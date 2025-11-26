# Data Handling

> Managing the complete lifecycle of Protected Health Information

---

## Overview

Proper data handling is essential for HIPAA compliance. This covers the **entire lifecycle** of PHI: collection, storage, usage, retention, and deletion. Backend developers must implement systems that protect PHI throughout its entire existence while respecting patient rights and regulatory requirements.

**Key Principle:** Handle PHI with the highest level of care, applying defense-in-depth strategies at every stage.

---

## HIPAA Requirements

### Minimum Necessary Standard (§ 164.502(b))

**Requirement:** Only use, disclose, and request the **minimum amount of PHI needed** to accomplish the intended purpose.

**Applies to:**

- Data collection (forms, APIs)
- Database queries
- API responses
- Data disclosures to third parties
- Internal access by employees

**Exceptions** (minimum necessary does NOT apply):

- Disclosures to healthcare providers for treatment
- Disclosures to the individual (the patient themselves)
- Uses/disclosures authorized by the patient
- Required by law

### Data Retention (§ 164.316(b)(2))

**Requirement:** Retain PHI and related documentation for **6 years** from creation or last effective date.

**Applies to:**

- Medical records
- Audit logs
- Security policies
- Training records
- BAAs (Business Associate Agreements)

### Secure Disposal (§ 164.310(d)(2)(i))

**Requirement:** Implement policies for final disposition of ePHI and the hardware/media on which it is stored.

**PHI must be rendered:**

- Unreadable
- Indecipherable
- Unrecoverable

---

## Data Minimization Principle

### Collect Only What's Necessary

**❌ Bad Example - Collecting unnecessary data:**

```typescript
/**
 * Over-collection of PHI
 */
interface PatientRegistration {
  // Required for appointment
  firstName: string;
  lastName: string;
  dateOfBirth: Date;
  email: string;
  phone: string;

  // ❌ Unnecessary for appointment booking
  ssn: string; // Not needed yet!
  mothersMaidenName: string; // Security question - not needed
  fullMedicalHistory: string; // Not needed at registration
  insuranceNumber: string; // Collect only when needed
  creditCardNumber: string; // Store in PCI-compliant system, not PHI database
}
```

**✅ Good Example - Minimum necessary:**

```typescript
/**
 * Collect minimum necessary for appointment
 */
interface AppointmentBooking {
  firstName: string;
  lastName: string;
  dateOfBirth: Date; // Verify identity
  email: string; // Confirmation
  phone: string; // Reminder
  appointmentType: string;
  preferredDate: Date;
}

/**
 * Collect additional PHI only when needed
 * e.g., during check-in or intake
 */
interface PatientIntake {
  // Extends AppointmentBooking with medical info
  medicalHistory: string; // Now necessary for treatment
  currentMedications: string[];
  allergies: string[];
  insuranceInfo?: InsuranceInfo; // Optional, if using insurance
}
```

### Database Queries - Minimum Necessary

**❌ Bad Example - Selecting all fields:**

```typescript
/**
 * Over-fetching PHI from database
 */
async getPatientForAppointment(patientId: string) {
  // ❌ Returns ALL patient data, including unnecessary PHI
  return await this.prisma.patient.findUnique({
    where: { id: patientId }
  });
}
```

**✅ Good Example - Select specific fields:**

```typescript
/**
 * Fetch only necessary fields for appointment
 */
async getPatientForAppointment(patientId: string) {
  // ✅ Select only fields needed for appointment context
  return await this.prisma.patient.findUnique({
    where: { id: patientId },
    select: {
      id: true,
      firstName: true,
      lastName: true,
      dateOfBirth: true,
      phone: true,
      // Exclude: ssn, address, medicalHistory, etc.
    }
  });
}

/**
 * Fetch full record only when treating
 */
async getPatientForTreatment(patientId: string, providerId: string) {
  // Verify provider has permission
  await this.verifyProviderAccess(providerId, patientId);

  // Log PHI access
  await this.auditLog.log({
    userId: providerId,
    action: AuditAction.PHI_READ,
    resource: 'patient',
    resourceId: patientId
  });

  // ✅ Full record justified for treatment
  return await this.prisma.patient.findUnique({
    where: { id: patientId }
  });
}
```

### API Responses - Minimum Necessary

```typescript
/**
 * DTO for appointment list (minimal PHI exposure)
 */
export class AppointmentListDto {
  id: string;
  patientInitials: string; // "J.D." instead of full name
  appointmentDate: Date;
  appointmentType: string;
  status: string;

  // ❌ Don't include in list view:
  // - Full patient name
  // - Date of birth
  // - Medical history
  // - Diagnosis codes
}

/**
 * DTO for full appointment details (more PHI justified)
 */
export class AppointmentDetailDto extends AppointmentListDto {
  patientName: string; // ✅ Now necessary
  dateOfBirth: Date; // ✅ Verify identity
  contactInfo: ContactInfo; // ✅ For communication
  medicalNotes: string; // ✅ For treatment
}
```

---

## Data Storage Patterns

### Database Schema Design

**Separate PHI from non-PHI tables:**

```prisma
/**
 * User account (non-PHI)
 * Can be queried freely
 */
model User {
  id        String   @id @default(uuid())
  email     String   @unique
  username  String   @unique
  role      Role
  isActive  Boolean  @default(true)

  // No PHI in this table!

  // Relation to PHI
  patient   Patient? @relation("UserPatient")

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@map("users")
}

/**
 * Patient PHI (sensitive)
 * Requires audit logging on all access
 */
model Patient {
  id             String   @id @default(uuid())

  // Demographic PHI
  firstName      String
  lastName       String
  dateOfBirth    DateTime
  gender         String?
  ssn            String?  // Nullable, encrypt if stored

  // Contact PHI
  phone          String?
  email          String?
  address        Address?

  // Medical PHI
  medicalHistory String?
  allergies      String[]
  medications    Medication[]
  appointments   Appointment[]

  // Audit fields
  createdAt      DateTime @default(now())
  createdBy      String
  updatedAt      DateTime @updatedAt
  updatedBy      String?

  // Relation to user account
  userId         String   @unique
  user           User     @relation("UserPatient", fields: [userId], references: [id])

  @@map("patients")
}
```

### Row-Level Security (PostgreSQL)

```sql
/**
 * PostgreSQL Row-Level Security (RLS)
 * Ensures users only access authorized records
 */

-- Enable RLS on patients table
ALTER TABLE patients ENABLE ROW LEVEL SECURITY;

-- Policy: Patients can only see their own record
CREATE POLICY patient_self_access ON patients
  FOR SELECT
  USING (user_id = current_setting('app.current_user_id')::UUID);

-- Policy: Providers can see patients they're assigned to
CREATE POLICY provider_patient_access ON patients
  FOR SELECT
  USING (
    id IN (
      SELECT patient_id
      FROM patient_provider_assignments
      WHERE provider_id = current_setting('app.current_user_id')::UUID
    )
  );

-- Policy: Only specific roles can insert
CREATE POLICY insert_patient ON patients
  FOR INSERT
  WITH CHECK (
    current_setting('app.current_user_role') IN ('provider', 'admin')
  );

-- Set user context before queries
-- In application code:
await prisma.$executeRaw`SET app.current_user_id = ${userId}`;
await prisma.$executeRaw`SET app.current_user_role = ${userRole}`;
```

---

## Data Deletion Requirements

### CRITICAL Lesson Learned: NO SOFT DELETES for PHI!

**❌ HIPAA VIOLATION - Soft Delete Pattern:**

```typescript
/**
 * ❌ WRONG: Soft delete keeps PHI in database
 * This violates HIPAA disposal requirements!
 */
model Patient {
  id        String   @id @default(uuid())
  firstName String
  lastName  String
  // ... other PHI fields

  deletedAt DateTime?  // ❌ Soft delete flag - NOT COMPLIANT!
  isDeleted Boolean @default(false)  // ❌ Also non-compliant!

  @@map("patients")
}

async deletePatient(id: string) {
  // ❌ WRONG: Data still exists in database
  return await this.prisma.patient.update({
    where: { id },
    data: { deletedAt: new Date(), isDeleted: true }
  });
}
```

**Why soft deletes violate HIPAA:**

1. PHI is not "unreadable, indecipherable, or unrecoverable"
2. Deleted data can be recovered with simple query
3. Increases data retention beyond necessary period
4. Creates unnecessary risk of exposure

### ✅ COMPLIANT: Hard Delete Pattern

```typescript
/**
 * ✅ CORRECT: Hard delete removes PHI permanently
 * Complies with HIPAA disposal requirements
 */
async deletePatient(id: string, userId: string, reason: string) {
  // 1. Verify deletion is authorized
  await this.verifyDeletionPermission(userId);

  // 2. Log BEFORE deletion (can't log after!)
  await this.auditLog.log({
    userId,
    action: AuditAction.PHI_DELETE,
    resource: 'patient',
    resourceId: id,
    details: {
      reason,  // Document why (e.g., "Patient requested deletion")
      deletedFields: ['firstName', 'lastName', 'dateOfBirth', 'ssn', 'medicalHistory'],
      timestamp: new Date()
    }
  });

  // 3. HARD DELETE - removes PHI from database
  await this.prisma.patient.delete({
    where: { id }
  });

  // 4. Audit log persists (contains no PHI, just record ID)
  // This is compliant - audit logs are exempt from deletion
}
```

### Cascade Deletions

```prisma
/**
 * Configure cascade deletes for related PHI
 */
model Patient {
  id           String        @id @default(uuid())
  // ... fields

  // All related PHI must be deleted together
  appointments Appointment[] @relation(onDelete: Cascade)
  medications  Medication[]  @relation(onDelete: Cascade)
  labResults   LabResult[]   @relation(onDelete: Cascade)

  @@map("patients")
}

model Appointment {
  id        String  @id @default(uuid())
  patientId String
  // ... fields

  patient   Patient @relation(fields: [patientId], references: [id], onDelete: Cascade)

  @@map("appointments")
}
```

### Deletion Workflow

```typescript
/**
 * Complete deletion workflow with safeguards
 */
@Injectable()
export class PatientDeletionService {
  constructor(
    private prisma: PrismaService,
    private auditLog: AuditLogService,
    private notificationService: NotificationService
  ) {}

  /**
   * Patient-requested deletion (Right to Erasure)
   * IMPORTANT: HIPAA does not require this, but state laws might (e.g., CCPA)
   */
  async requestDeletion(patientId: string, requestedBy: string) {
    // 1. Create deletion request (for approval)
    const request = await this.prisma.deletionRequest.create({
      data: {
        patientId,
        requestedBy,
        requestedAt: new Date(),
        status: "PENDING",
        reason: "Patient requested deletion",
      },
    });

    // 2. Notify privacy officer for review
    await this.notificationService.notifyPrivacyOfficer(request);

    return request;
  }

  /**
   * Execute approved deletion
   */
  async executeDeletion(requestId: string, approvedBy: string) {
    const request = await this.prisma.deletionRequest.findUnique({
      where: { id: requestId },
    });

    if (!request || request.status !== "APPROVED") {
      throw new Error("Deletion request not approved");
    }

    // Begin transaction - all or nothing
    await this.prisma.$transaction(async (tx) => {
      // 1. Audit log BEFORE deletion
      await this.auditLog.log({
        userId: approvedBy,
        action: AuditAction.PHI_DELETE,
        resource: "patient",
        resourceId: request.patientId,
        details: {
          requestId,
          reason: request.reason,
          approvedBy,
        },
      });

      // 2. Hard delete patient and all related PHI
      // Cascade deletes appointments, medications, lab results, etc.
      await tx.patient.delete({
        where: { id: request.patientId },
      });

      // 3. Mark deletion request as completed
      await tx.deletionRequest.update({
        where: { id: requestId },
        data: {
          status: "COMPLETED",
          completedAt: new Date(),
          completedBy: approvedBy,
        },
      });
    });

    // 4. Notify requester
    await this.notificationService.notifyDeletionComplete(request);
  }
}
```

### What NOT to Delete

**Audit logs are EXEMPT from deletion:**

- Audit logs must be retained for 6 years
- Even if patient is deleted, audit trail remains
- Audit logs should contain NO PHI (only IDs)

**Legal holds:**

- PHI involved in litigation must be preserved
- Implement legal hold flags in database

```typescript
model Patient {
  id         String  @id @default(uuid())
  // ... PHI fields

  legalHold  Boolean @default(false)  // Prevent deletion if true
  holdReason String?
  holdUntil  DateTime?

  @@map("patients")
}

async deletePatient(id: string, userId: string) {
  const patient = await this.prisma.patient.findUnique({ where: { id } });

  // Prevent deletion if legal hold
  if (patient.legalHold) {
    throw new ForbiddenException(
      `Cannot delete patient under legal hold: ${patient.holdReason}`
    );
  }

  // Proceed with deletion
  // ...
}
```

---

## Data Retention Policies

### HIPAA Requirement: 6-Year Retention

**§ 164.316(b)(2)(i):** Retain documentation for 6 years from creation or last effective date.

### Implementation Strategy

```typescript
/**
 * Automated data retention policy
 */
interface RetentionPolicy {
  dataType: string;
  retentionPeriod: number;  // Years
  deletionMethod: 'hard_delete' | 'archive';
}

const RETENTION_POLICIES: RetentionPolicy[] = [
  {
    dataType: 'patient_records',
    retentionPeriod: 7,  // 6 years + 1 buffer
    deletionMethod: 'hard_delete'
  },
  {
    dataType: 'audit_logs',
    retentionPeriod: 6,
    deletionMethod: 'archive'  // Move to cold storage, don't delete
  },
  {
    dataType: 'appointments',
    retentionPeriod: 7,
    deletionMethod: 'hard_delete'
  }
];

/**
 * Scheduled job to enforce retention policies
 * Runs monthly
 */
@Cron('0 0 1 * *')  // 1st day of month at midnight
async enforceRetentionPolicies() {
  for (const policy of RETENTION_POLICIES) {
    const cutoffDate = new Date();
    cutoffDate.setFullYear(cutoffDate.getFullYear() - policy.retentionPeriod);

    if (policy.deletionMethod === 'hard_delete') {
      // Find records older than retention period
      const expiredRecords = await this.prisma[policy.dataType].findMany({
        where: {
          createdAt: { lt: cutoffDate },
          legalHold: false  // Don't delete if legal hold
        },
        select: { id: true }
      });

      // Log bulk deletion
      await this.auditLog.log({
        userId: 'SYSTEM',
        action: AuditAction.PHI_DELETE,
        resource: policy.dataType,
        details: {
          reason: 'Automated retention policy',
          recordCount: expiredRecords.length,
          cutoffDate
        }
      });

      // Hard delete expired records
      await this.prisma[policy.dataType].deleteMany({
        where: {
          id: { in: expiredRecords.map(r => r.id) }
        }
      });
    } else if (policy.deletionMethod === 'archive') {
      // Archive to S3 Glacier (for audit logs)
      await this.archiveToGlacier(policy.dataType, cutoffDate);
    }
  }
}
```

---

## Backup and Recovery

### RDS Automated Backups

```typescript
/**
 * RDS configuration with automated backups
 */
const database = new rds.DatabaseInstance(stack, "Database", {
  engine: rds.DatabaseInstanceEngine.postgres({
    version: rds.PostgresEngineVersion.VER_15_3,
  }),

  // HIPAA: Encrypted backups
  storageEncrypted: true,
  storageEncryptionKey: kmsKey,

  // Backup configuration
  backupRetention: cdk.Duration.days(30), // 30-day retention
  deleteAutomatedBackups: false, // Keep backups after instance deletion
  preferredBackupWindow: "03:00-04:00", // Low-traffic window

  // Point-in-time recovery
  multiAz: true, // High availability
  allocatedStorage: 100,
  maxAllocatedStorage: 500, // Auto-scaling
});
```

### Backup Testing

**HIPAA requires periodic testing of backup restoration:**

```typescript
/**
 * Quarterly backup restore test
 * Required to verify backups are functional
 */
@Cron('0 0 1 */3 *')  // Every 3 months
async testBackupRestore() {
  // 1. Create test snapshot
  const snapshot = await this.rds.createDBSnapshot({
    DBSnapshotIdentifier: `test-restore-${Date.now()}`,
    DBInstanceIdentifier: 'production-db'
  });

  // 2. Restore to test instance
  const testInstance = await this.rds.restoreDBInstanceFromDBSnapshot({
    DBInstanceIdentifier: 'test-restore-instance',
    DBSnapshotIdentifier: snapshot.DBSnapshot.DBSnapshotIdentifier,
    PubliclyAccessible: false,
    VpcSecurityGroupIds: ['test-security-group']
  });

  // 3. Verify data integrity
  const isValid = await this.verifyDatabaseIntegrity(testInstance);

  // 4. Log test results
  await this.auditLog.log({
    userId: 'SYSTEM',
    action: 'BACKUP_TEST',
    status: isValid ? 'success' : 'failure',
    details: {
      snapshotId: snapshot.DBSnapshot.DBSnapshotIdentifier,
      testResult: isValid ? 'PASSED' : 'FAILED'
    }
  });

  // 5. Clean up test instance
  await this.rds.deleteDBInstance({
    DBInstanceIdentifier: 'test-restore-instance',
    SkipFinalSnapshot: true
  });

  if (!isValid) {
    throw new Error('Backup restore test failed!');
  }
}
```

---

## PostgreSQL Best Practices

### Partitioning for Performance

```sql
/**
 * Partition large tables by date for better performance
 * Useful for audit logs and historical data
 */

CREATE TABLE audit_logs (
  id UUID,
  timestamp TIMESTAMP NOT NULL,
  user_id UUID,
  action VARCHAR(50),
  -- ... other fields
) PARTITION BY RANGE (timestamp);

-- Create partitions for each month
CREATE TABLE audit_logs_2025_01 PARTITION OF audit_logs
  FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE audit_logs_2025_02 PARTITION OF audit_logs
  FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

-- Automate partition creation with function
CREATE OR REPLACE FUNCTION create_monthly_partition()
RETURNS void AS $$
DECLARE
  start_date DATE;
  end_date DATE;
  partition_name TEXT;
BEGIN
  start_date := date_trunc('month', CURRENT_DATE + INTERVAL '1 month');
  end_date := start_date + INTERVAL '1 month';
  partition_name := 'audit_logs_' || to_char(start_date, 'YYYY_MM');

  EXECUTE format(
    'CREATE TABLE IF NOT EXISTS %I PARTITION OF audit_logs FOR VALUES FROM (%L) TO (%L)',
    partition_name, start_date, end_date
  );
END;
$$ LANGUAGE plpgsql;
```

### Vacuum and Maintenance

```sql
/**
 * Regular maintenance for PostgreSQL performance
 * Especially important after hard deletes
 */

-- Analyze tables to update statistics
ANALYZE patients;
ANALYZE appointments;

-- Vacuum to reclaim space from deleted rows
VACUUM FULL patients;  -- Reclaims maximum space but locks table

-- Auto-vacuum settings (postgresql.conf)
autovacuum = on
autovacuum_vacuum_scale_factor = 0.1
autovacuum_analyze_scale_factor = 0.05
```

### Connection Pooling

```typescript
/**
 * Prisma connection pooling
 * Efficient database connections
 */
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")

  // Connection pool settings
  connectionLimit = 10  // Max connections per instance
}

// .env
DATABASE_URL="postgresql://user:pass@host:5432/db?connection_limit=10&pool_timeout=30"
```

---

## Common Mistakes & How to Avoid Them

### ❌ Mistake 1: Soft Deletes for PHI

**Problem:** Using `deletedAt` flags instead of hard deletes
**HIPAA Violation:** § 164.310(d)(2)(i) - Data not truly disposed
**Solution:** Implement hard delete with audit trail

### ❌ Mistake 2: Over-Collection of PHI

**Problem:** Collecting SSN, full medical history at registration
**HIPAA Violation:** § 164.502(b) - Minimum necessary
**Solution:** Collect only what's needed for current purpose

### ❌ Mistake 3: Fetching Full Records Unnecessarily

**Problem:** `SELECT * FROM patients` for appointment list
**HIPAA Risk:** Unnecessary PHI exposure
**Solution:** Use Prisma `select` to fetch only needed fields

### ❌ Mistake 4: No Retention Policy

**Problem:** Keeping all data forever
**HIPAA Risk:** Increased attack surface, storage costs
**Solution:** Automated retention policy with hard deletes after 6-7 years

### ❌ Mistake 5: Untested Backups

**Problem:** Never verifying backups can be restored
**HIPAA Risk:** Data loss during incident
**Solution:** Quarterly backup restore tests

### ❌ Mistake 6: No Cascade Deletes

**Problem:** Deleting patient but leaving orphaned PHI in related tables
**HIPAA Violation:** PHI not fully disposed
**Solution:** Configure `onDelete: Cascade` in Prisma schema

---

## Data Handling Decision Tree

Use the following decision flow when deciding how to handle a particular data element in your system:

```mermaid
flowchart TD
  A[Start: New/Existing Data Element] --> B{Is it PHI or can it<br/>become PHI when combined<br/>with other data?}
  B -- No --> C[Store as non-PHI<br/>with standard security]
  B -- Yes --> D{Is this data<br/>necessary for the<br/>current purpose?}
  D -- No --> E[Do not collect<br/>or delete if already stored]
  D -- Yes --> F{Is there a legal<br/>or contractual<br/>retention requirement?}
  F -- No --> G[Apply internal retention<br/>policy (e.g., 1-3 years)]
  F -- Yes --> H[Apply HIPAA/state<br/>retention rules<br/>(typically ≥ 6 years)]
  G --> I{Is this PHI still needed<br/>for treatment, payment,<br/>or operations?}
  H --> I
  I -- Yes --> J[Keep in primary database<br/>with full protections]
  I -- No --> K{Is a legal hold<br/>in place?}
  K -- Yes --> L[Keep until hold<br/>is released]
  K -- No --> M[Hard delete from primary DB;<br/>keep non-PHI audit trail]
  C --> N[End]
  E --> N
  J --> N
  L --> N
  M --> N
```

### Practical Mapping to Our Stack

- **New field in Prisma model**
  - Ask: _Can this field identify a patient directly or indirectly?_
  - If yes → treat as PHI, apply RLS, audit logging, encryption, and retention rules.
- **Existing column no longer used**
  - If contains PHI and no legal/contractual need → schedule **hard delete** via retention job.
- **Derived/analytics data**
  - Prefer aggregated/de-identified views.
  - If re-identification is possible → treat as PHI and follow full lifecycle.
- **Backups and archives**
  - Must follow the same retention rules as primary data.
  - When retention expires → rotate out encrypted snapshots/archives per DR policy.

---

## Checklist

Before deploying data handling systems:

- [ ] **Minimum necessary implemented** - Only collect/query needed PHI
- [ ] **Hard delete required** - No soft delete flags for PHI
- [ ] **Cascade deletes configured** - Related PHI deleted together
- [ ] **Deletion workflow** - Audit log before delete
- [ ] **Legal hold flags** - Prevent deletion of litigation-related PHI
- [ ] **Retention policies** - Automated deletion after 6-7 years
- [ ] **Backup encryption** - RDS backups encrypted with KMS
- [ ] **Backup testing** - Quarterly restore tests
- [ ] **Row-level security** - PostgreSQL RLS policies
- [ ] **Partitioning** - Large tables partitioned by date
- [ ] **Vacuum scheduled** - Regular maintenance after deletes
- [ ] **Connection pooling** - Efficient database connections
- [ ] **Audit logs exempt** - Audit logs retained separately

---

## Additional Resources

**HIPAA Guidance:**

- [HHS Disposal of PHI FAQ](https://www.hhs.gov/hipaa/for-professionals/faq/disposal-of-protected-health-information/index.html)
- [HIPAA Minimum Necessary Rule](https://www.hipaajournal.com/ahima-hipaa-minimum-necessary-standard-3481/)
- [Secure Data Deletion](https://www.securitymetrics.com/blog/secure-data-deletion-permanently-deleting-phi-healthcare)

**Standards:**

- [NIST SP 800-88 - Media Sanitization](https://csrc.nist.gov/publications/detail/sp/800-88/rev-1/final)

**PostgreSQL:**

- [Row-Level Security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- [Table Partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html)

---

## Next Steps

Continue to the final section:

**[→ Backend Checklist](05-checklist.md)**

Or review related topics:

- [Audit Logging](03-audit-logging.md) - Log deletion events
- [Encryption](02-encryption.md) - Protect stored PHI

---

_Last Updated: November 2025_
_Part of the HIPAA Compliance Handbook - Backend Playbook_
