---
layout: default
title: Encryption
parent: Backend Playbook
nav_order: 3
---

# Encryption

> Protecting ePHI at rest and in transit with encryption

---

## Overview

Encryption is a **mandatory technical safeguard** under the 2025 HIPAA Security Rule updates. All electronic Protected Health Information (ePHI) must be encrypted both **at rest** (stored data) and **in transit** (data being transmitted).

**Before 2025:** Encryption was "addressable" (implement or document why not)
**After 2025:** Encryption is **REQUIRED** with no exceptions

---

## HIPAA Requirements

### Encryption and Decryption (§ 164.312(a)(2)(iv))

**2025 Status:** **REQUIRED** (previously addressable)

**Scope:**

- All ePHI must be encrypted at rest
- All ePHI must be encrypted in transit
- Encryption keys must be properly managed
- Decryption must be logged and audited

### Transmission Security (§ 164.312(e)(1))

**Required:** Technical security measures to guard against unauthorized access to ePHI transmitted over electronic networks.

**Includes:**

- Encryption of data in transit
- Secure communication channels (TLS/HTTPS)
- End-to-end encryption where applicable

---

## 2025 Encryption Standards

The 2025 HIPAA updates specify exact encryption standards:

### Data at Rest

- **Standard:** AES-256 (Advanced Encryption Standard)
- **Minimum:** AES-256 is the baseline requirement
- **Certification:** FIPS 140-2 Level 2 minimum (Level 3 recommended for high-risk)

### Data in Transit

- **Standard:** TLS 1.3 (Transport Layer Security)
- **Minimum:** TLS 1.2 or higher
- **No longer acceptable:** TLS 1.0, TLS 1.1, SSL (all deprecated)

### Key Exchanges

- **Standard:** RSA-4096 or ECC (Elliptic Curve Cryptography)
- **Minimum:** RSA-2048
- **Not acceptable:** RSA-1024 or weaker

### Compliance Deadline

**December 31, 2025** - All healthcare organizations must comply with new encryption standards

---

## Data at Rest Encryption

### PostgreSQL / Amazon RDS Encryption

**AWS RDS** provides built-in encryption at rest using AWS Key Management Service (KMS):

#### Enabling Encryption

```typescript
/**
 * RDS configuration with encryption enabled
 * Note: Must be enabled at DB instance creation - cannot be added later!
 */
const dbInstance = new rds.DatabaseInstance(stack, "Database", {
  engine: rds.DatabaseInstanceEngine.postgres({
    version: rds.PostgresEngineVersion.VER_15_3,
  }),
  instanceType: ec2.InstanceType.of(
    ec2.InstanceClass.T3,
    ec2.InstanceSize.MEDIUM
  ),

  // HIPAA Requirement: Encryption at rest
  storageEncrypted: true, // REQUIRED for HIPAA
  storageEncryptionKey: kmsKey, // Customer-managed KMS key

  // SSL/TLS required for connections
  enableIamDatabaseAuthentication: false, // Use username/password with SSL

  // Backup encryption (inherits from instance)
  backupRetention: cdk.Duration.days(30),

  // Network isolation
  vpc: vpc,
  vpcSubnets: {
    subnetType: ec2.SubnetType.PRIVATE_ISOLATED, // No internet access
  },
});
```

#### Key Features of RDS Encryption:

✅ **Encrypted Storage** - All data at rest (database, backups, snapshots, read replicas)
✅ **Automatic Encryption** - Transparent to application (no code changes)
✅ **KMS Integration** - Keys managed by AWS KMS
✅ **FIPS 140-2 Level 2** - Compliant encryption modules

**Important Limitations:**

- ⚠️ **Cannot enable encryption on existing unencrypted DB** - Must create new encrypted instance and migrate
- ⚠️ **Cannot change KMS key** after creation - Choose carefully
- ⚠️ **Cannot disable encryption** once enabled

#### AWS KMS Key Configuration

```typescript
/**
 * Create Customer Managed KMS Key for RDS encryption
 * Best practice: Separate key per database/environment
 */
const rdsKmsKey = new kms.Key(stack, "RDSEncryptionKey", {
  description: "KMS key for RDS database encryption (HIPAA)",
  enableKeyRotation: true, // REQUIRED: Automatic annual rotation

  // Key policy - least privilege access
  policy: new iam.PolicyDocument({
    statements: [
      // Allow root account to manage key
      new iam.PolicyStatement({
        sid: "Enable IAM User Permissions",
        principals: [new iam.AccountRootPrincipal()],
        actions: ["kms:*"],
        resources: ["*"],
      }),
      // Allow RDS service to use key
      new iam.PolicyStatement({
        sid: "Allow RDS to use the key",
        principals: [new iam.ServicePrincipal("rds.amazonaws.com")],
        actions: ["kms:Decrypt", "kms:DescribeKey", "kms:CreateGrant"],
        resources: ["*"],
      }),
    ],
  }),

  // Deletion protection
  removalPolicy: cdk.RemovalPolicy.RETAIN, // Prevent accidental deletion
  pendingWindow: cdk.Duration.days(30), // 30-day recovery window
});

// Alias for easier identification
rdsKmsKey.addAlias("alias/rds-database-hipaa");
```

#### PostgreSQL Force SSL Configuration

```sql
-- Set in RDS Parameter Group
-- This MUST be set to enforce encrypted connections

-- Force SSL for all connections (HIPAA REQUIRED)
rds.force_ssl = 1

-- Alternative: Set in postgresql.conf for self-hosted
ssl = on
ssl_ciphers = 'HIGH:!aNULL:!MD5'
ssl_min_protocol_version = 'TLSv1.2'
```

#### Connection String with SSL

```typescript
/**
 * PostgreSQL connection with SSL enforced
 * Prisma example
 */
// .env
DATABASE_URL =
  "postgresql://user:password@hostname:5432/dbname?sslmode=require";

// For AWS RDS, download RDS CA certificate
DATABASE_URL =
  "postgresql://user:password@hostname:5432/dbname?sslmode=verify-full&sslrootcert=./rds-ca-2019-root.pem";
```

```typescript
/**
 * TypeORM connection with SSL
 */
const dataSource = new DataSource({
  type: "postgres",
  host: process.env.DB_HOST,
  port: 5432,
  username: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,

  // HIPAA REQUIRED: SSL/TLS encryption
  ssl: {
    rejectUnauthorized: true, // Verify server certificate
    ca: fs.readFileSync("./rds-ca-2019-root.pem").toString(),
  },
});
```

---

## Column-Level Encryption: When Is It Needed?

### Lesson Learned: AWS KMS Encryption Is Sufficient

For most HIPAA use cases, **AWS RDS encryption with KMS is sufficient**. Column-level encryption adds significant complexity with minimal security benefit when full-disk encryption is already in place.

### When AWS RDS Encryption Is Enough ✅

**Use RDS/KMS encryption when:**

- All database access is through your application
- Database admins don't need to see raw data
- You have proper access controls and audit logging
- Data is in a HIPAA-eligible AWS service

**Benefits:**

- ✅ Transparent to application (no code changes)
- ✅ Protects against physical theft of storage
- ✅ Encrypts backups and snapshots automatically
- ✅ No performance overhead from app-level encryption
- ✅ Simpler to implement and maintain

### When Column-Level Encryption Is Needed ⚠️

**Consider column-level encryption when:**

- Database administrators should not see certain fields (e.g., SSN, credit cards)
- Compliance requires field-level encryption (e.g., PCI DSS for card numbers)
- You need different keys for different data fields
- You need to prove specific fields are encrypted separately

**Drawbacks:**

- ❌ Cannot query encrypted fields (no WHERE clause on encrypted data)
- ❌ Cannot index encrypted columns (slow queries)
- ❌ Requires encryption/decryption in application (performance overhead)
- ❌ Key management complexity
- ❌ More code to maintain

### Implementation Example (If Needed)

```typescript
import * as crypto from "crypto";

/**
 * Column-level encryption service using AWS KMS
 * Only use if RDS encryption is insufficient!
 */
@Injectable()
export class EncryptionService {
  constructor(private kmsClient: KMSClient) {}

  /**
   * Encrypt sensitive field with AWS KMS
   */
  async encrypt(plaintext: string, keyId: string): Promise<string> {
    const command = new EncryptCommand({
      KeyId: keyId,
      Plaintext: Buffer.from(plaintext, "utf-8"),
    });

    const response = await this.kmsClient.send(command);

    // Return base64-encoded ciphertext
    return Buffer.from(response.CiphertextBlob).toString("base64");
  }

  /**
   * Decrypt sensitive field
   */
  async decrypt(ciphertext: string): Promise<string> {
    const command = new DecryptCommand({
      CiphertextBlob: Buffer.from(ciphertext, "base64"),
    });

    const response = await this.kmsClient.send(command);

    return Buffer.from(response.Plaintext).toString("utf-8");
  }
}
```

**Database Schema with Encrypted Fields:**

```typescript
/**
 * Prisma model with encrypted field
 * Note: Cannot query on encrypted fields!
 */
model Patient {
  id        String   @id @default(uuid())
  firstName String   // Unencrypted (searchable)
  lastName  String   // Unencrypted (searchable)

  // Encrypted column (use only if RDS encryption insufficient)
  ssnEncrypted String @map("ssn_encrypted")  // Cannot search on this!

  @@map("patients")
}
```

**⚠️ Important:** We determined column-level encryption was **NOT necessary** for our HIPAA project. RDS encryption with KMS was sufficient.

---

## Data in Transit Encryption

### TLS/HTTPS Requirements

All communication containing ePHI must use **TLS 1.2 or higher**:

#### API Gateway / Load Balancer Configuration

```typescript
/**
 * Application Load Balancer with TLS 1.2+ enforcement
 */
const alb = new elbv2.ApplicationLoadBalancer(stack, "ALB", {
  vpc: vpc,
  internetFacing: true,

  // HIPAA: Drop invalid HTTP headers
  dropInvalidHeaderFields: true,
});

// HTTPS Listener with TLS 1.2+
const httpsListener = alb.addListener("HttpsListener", {
  port: 443,
  protocol: elbv2.ApplicationProtocol.HTTPS,

  // SSL Certificate from AWS Certificate Manager
  certificates: [certificate],

  // HIPAA REQUIRED: TLS 1.2+ only
  sslPolicy: elbv2.SslPolicy.TLS12_EXT, // TLS 1.2 minimum

  defaultAction: elbv2.ListenerAction.forward([targetGroup]),
});

// Redirect HTTP to HTTPS (never allow unencrypted)
alb.addListener("HttpListener", {
  port: 80,
  protocol: elbv2.ApplicationProtocol.HTTP,
  defaultAction: elbv2.ListenerAction.redirect({
    protocol: "HTTPS",
    port: "443",
    permanent: true,
  }),
});
```

#### AWS Certificate Manager (ACM)

```typescript
/**
 * Request SSL/TLS certificate from ACM
 * Free and auto-renewing
 */
const certificate = new acm.Certificate(stack, "Certificate", {
  domainName: "api.example.com",
  subjectAlternativeNames: ["*.api.example.com"],
  validation: acm.CertificateValidation.fromDns(hostedZone),
});
```

**Benefits of ACM:**

- ✅ Free SSL/TLS certificates
- ✅ Automatic renewal (no expiration issues)
- ✅ FIPS 140-2 Level 2 compliant
- ✅ Integration with ALB, CloudFront, API Gateway

#### NestJS HTTPS Enforcement

```typescript
/**
 * Helmet middleware for security headers
 * Enforces HTTPS, prevents downgrade attacks
 */
import helmet from "helmet";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Security headers
  app.use(
    helmet({
      hsts: {
        maxAge: 31536000, // 1 year
        includeSubDomains: true,
        preload: true,
      },
      contentSecurityPolicy: {
        directives: {
          defaultSrc: ["'self'"],
          upgradeInsecureRequests: [], // Force HTTPS
        },
      },
    })
  );

  // Trust proxy (for ALB)
  app.set("trust proxy", 1);

  await app.listen(3000);
}
```

#### API Client Configuration (Frontend)

```typescript
/**
 * Axios client with HTTPS enforcement
 */
import axios from "axios";

const apiClient = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL, // Must be HTTPS in production

  // HIPAA: Only allow HTTPS
  httpsAgent: new https.Agent({
    rejectUnauthorized: true, // Verify SSL certificate
    minVersion: "TLSv1.2", // Minimum TLS 1.2
  }),

  timeout: 30000,
  headers: {
    "Content-Type": "application/json",
  },
});

// Reject requests over HTTP
apiClient.interceptors.request.use((config) => {
  if (config.url && config.url.startsWith("http://")) {
    throw new Error("HIPAA violation: HTTP not allowed, use HTTPS");
  }
  return config;
});
```

---

## Key Management with AWS KMS

### KMS Best Practices

#### 1. Customer Managed Keys (CMKs)

**Always use Customer Managed Keys for HIPAA workloads:**

```typescript
/**
 * Customer Managed KMS Key
 * Required for HIPAA compliance
 */
const kmsKey = new kms.Key(stack, "HIPAAKey", {
  description: "Customer managed key for HIPAA workloads",

  // REQUIRED: Automatic key rotation every year
  enableKeyRotation: true,

  // Key administrators (can manage but not use key)
  admins: [adminRole],

  // Deletion protection
  removalPolicy: cdk.RemovalPolicy.RETAIN,
  pendingWindow: cdk.Duration.days(30), // 30-day recovery period
});
```

**Why Customer Managed Keys?**

- ✅ Full control over key policies and permissions
- ✅ Required for audit trails and compliance
- ✅ Can grant cross-account access if needed
- ✅ Automatic rotation support
- ✅ CloudTrail logs all key usage

**AWS-Managed Keys limitations:**

- ❌ No rotation control
- ❌ Limited visibility in audit logs
- ❌ Cannot customize key policies
- ❌ Not suitable for HIPAA compliance

#### 2. Key Rotation Policy

**HIPAA Requirement:** Encryption keys must be rotated regularly

```typescript
/**
 * KMS key with automatic annual rotation
 */
enableKeyRotation: true; // Rotates every 365 days automatically
```

**Manual Rotation (Advanced):**
For more control, create new key version and update references:

```typescript
/**
 * Manual key rotation strategy
 * 1. Create new key
 * 2. Update applications to use new key for encryption
 * 3. Keep old key for decrypting existing data
 * 4. Eventually re-encrypt all data with new key
 */
async rotateKeys() {
  // Create new key version
  const newKey = await this.createNewKMSKey();

  // Update application config
  await this.updateKeyReference(newKey.keyId);

  // Schedule re-encryption job
  await this.scheduleReEncryption(oldKeyId, newKey.keyId);
}
```

#### 3. Key Policies (Least Privilege)

```typescript
/**
 * KMS key policy - least privilege access
 */
const keyPolicy = new iam.PolicyDocument({
  statements: [
    // Root account (for key management)
    new iam.PolicyStatement({
      sid: "Enable IAM User Permissions",
      principals: [new iam.AccountRootPrincipal()],
      actions: ["kms:*"],
      resources: ["*"],
    }),

    // Allow specific service (RDS) to use key
    new iam.PolicyStatement({
      sid: "Allow RDS Service",
      principals: [new iam.ServicePrincipal("rds.amazonaws.com")],
      actions: ["kms:Decrypt", "kms:DescribeKey", "kms:CreateGrant"],
      resources: ["*"],
      conditions: {
        StringEquals: {
          "kms:ViaService": [`rds.${stack.region}.amazonaws.com`],
        },
      },
    }),

    // Allow application role to decrypt only
    new iam.PolicyStatement({
      sid: "Allow Application Decryption",
      principals: [appRole],
      actions: ["kms:Decrypt", "kms:DescribeKey"],
      resources: ["*"],
    }),
  ],
});
```

#### 4. Monitoring Key Usage

**CloudTrail logs all KMS API calls:**

```typescript
/**
 * CloudWatch alarm for unauthorized KMS access attempts
 */
const unauthorizedKMSAttempts = new logs.MetricFilter(
  stack,
  "UnauthorizedKMS",
  {
    logGroup: cloudtrailLogGroup,
    filterPattern: logs.FilterPattern.all(
      logs.FilterPattern.stringValue("$.eventName", "=", "Decrypt"),
      logs.FilterPattern.stringValue("$.errorCode", "=", "AccessDenied")
    ),
    metricNamespace: "HIPAA/Security",
    metricName: "UnauthorizedKMSAccess",
    metricValue: "1",
  }
);

// Alarm on unauthorized attempts
new cloudwatch.Alarm(stack, "KMSAccessAlarm", {
  metric: unauthorizedKMSAttempts.metric(),
  threshold: 1,
  evaluationPeriods: 1,
  alarmDescription: "Unauthorized KMS key access attempt detected",
});
```

---

## Backup Encryption

### RDS Automated Backups

**RDS encrypted instances automatically encrypt backups:**

```typescript
/**
 * RDS backup configuration
 * Backups inherit encryption from instance
 */
backupRetention: cdk.Duration.days(30),  // HIPAA: 30+ days recommended
deleteAutomatedBackups: false,  // Retain backups after instance deletion
preferredBackupWindow: '03:00-04:00',  // Low-traffic window
```

**Important:**

- ✅ Automated backups are encrypted with same KMS key as instance
- ✅ Manual snapshots must be explicitly copied with encryption
- ⚠️ **Cannot share encrypted snapshots** without granting KMS key access

### Manual Snapshot Encryption

```typescript
/**
 * Copy unencrypted snapshot to encrypted snapshot
 */
const copySnapshot = new rds.CfnDBSnapshot(stack, "EncryptedSnapshot", {
  dbSnapshotIdentifier: "encrypted-snapshot",
  sourceDbSnapshotIdentifier: "unencrypted-snapshot",

  // Encrypt copy with KMS
  kmsKeyId: kmsKey.keyArn,
});
```

### S3 Backup Encryption

```typescript
/**
 * S3 bucket for encrypted backups
 */
const backupBucket = new s3.Bucket(stack, "BackupBucket", {
  bucketName: "hipaa-backups",

  // HIPAA REQUIRED: Encryption at rest
  encryption: s3.BucketEncryption.KMS,
  encryptionKey: kmsKey,

  // Enforce encryption for all uploads
  enforceSSL: true, // Require HTTPS

  // Block public access
  blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,

  // Versioning for data integrity
  versioned: true,

  // Lifecycle policy
  lifecycleRules: [
    {
      id: "DeleteOldBackups",
      expiration: cdk.Duration.days(2555), // 7 years (HIPAA retention)
      transitions: [
        {
          storageClass: s3.StorageClass.GLACIER,
          transitionAfter: cdk.Duration.days(90), // Archive after 90 days
        },
      ],
    },
  ],
});
```

---

## Testing Encryption

### Verify RDS Encryption

```typescript
/**
 * Integration test: Verify database encryption
 */
describe("Database Encryption", () => {
  it("should have encryption enabled", async () => {
    const dbInstance = await rds.describeDBInstances({
      DBInstanceIdentifier: "my-db-instance",
    });

    expect(dbInstance.DBInstances[0].StorageEncrypted).toBe(true);
    expect(dbInstance.DBInstances[0].KmsKeyId).toBeDefined();
  });

  it("should enforce SSL connections", async () => {
    // Attempt connection without SSL
    const connectionWithoutSSL = {
      host: process.env.DB_HOST,
      port: 5432,
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      ssl: false,
    };

    // Should fail if rds.force_ssl = 1
    await expect(new Client(connectionWithoutSSL).connect()).rejects.toThrow();
  });
});
```

### Verify TLS/HTTPS

```typescript
/**
 * Test: Ensure HTTPS is enforced
 */
describe("HTTPS Enforcement", () => {
  it("should reject HTTP requests", async () => {
    const httpUrl = process.env.API_URL.replace("https://", "http://");

    await expect(axios.get(httpUrl + "/health")).rejects.toThrow();
  });

  it("should use TLS 1.2+", async () => {
    const response = await axios.get(process.env.API_URL + "/health", {
      httpsAgent: new https.Agent({
        minVersion: "TLSv1.2",
      }),
    });

    expect(response.status).toBe(200);
  });

  it("should redirect HTTP to HTTPS", async () => {
    const response = await axios.get(
      process.env.API_URL.replace("https://", "http://") + "/health",
      { maxRedirects: 0, validateStatus: () => true }
    );

    expect(response.status).toBe(301); // Permanent redirect
    expect(response.headers.location).toMatch(/^https:\/\//);
  });
});
```

---

## Common Mistakes & How to Avoid Them

### ❌ Mistake 1: Unencrypted Database

**Problem:** Creating RDS instance without encryption
**Solution:** Always set `storageEncrypted: true` at creation (cannot be added later!)

### ❌ Mistake 2: Using AWS-Managed Keys

**Problem:** Using default AWS-managed KMS keys for HIPAA workloads
**Solution:** Always create Customer Managed Keys with rotation enabled

### ❌ Mistake 3: No Key Rotation

**Problem:** KMS keys never rotated
**Solution:** Enable automatic rotation (`enableKeyRotation: true`)

### ❌ Mistake 4: TLS 1.0 / 1.1 Allowed

**Problem:** Load balancer allows deprecated TLS versions
**Solution:** Set `sslPolicy: elbv2.SslPolicy.TLS12_EXT` (TLS 1.2+)

### ❌ Mistake 5: HTTP Allowed

**Problem:** API accepts both HTTP and HTTPS
**Solution:** Redirect all HTTP to HTTPS, reject HTTP at application level

### ❌ Mistake 6: Database Connections Without SSL

**Problem:** `sslmode=disable` in connection string
**Solution:** Always use `sslmode=require` or `sslmode=verify-full`

### ❌ Mistake 7: Unencrypted Backups

**Problem:** Manual backups not encrypted
**Solution:** Always encrypt snapshots, verify S3 bucket encryption

---

## Checklist

Before deploying backend with ePHI:

- [ ] **RDS encryption enabled** - `storageEncrypted: true` with Customer Managed Key
- [ ] **KMS key rotation enabled** - Automatic annual rotation
- [ ] **Database SSL enforced** - `rds.force_ssl = 1` parameter set
- [ ] **Connection strings use SSL** - `sslmode=require` minimum
- [ ] **TLS 1.2+ on load balancer** - `SslPolicy.TLS12_EXT`
- [ ] **HTTP redirects to HTTPS** - No unencrypted endpoints
- [ ] **ACM certificate configured** - Valid SSL/TLS certificate
- [ ] **HSTS headers enabled** - `Strict-Transport-Security` header
- [ ] **S3 backups encrypted** - KMS encryption with Customer Managed Key
- [ ] **Key policies follow least privilege** - Minimal permissions granted
- [ ] **CloudTrail logging KMS usage** - All key access logged
- [ ] **No column-level encryption** - Unless specifically required (RDS encryption sufficient)

---

## Additional Resources

**HIPAA Guidance:**

- [HHS Encryption FAQ](https://www.hhs.gov/hipaa/for-professionals/faq/encryption/index.html)
- [HIPAA Encryption Requirements 2025](https://www.hipaajournal.com/hipaa-encryption-requirements/)
- [2025 Encryption Updates](https://www.censinet.com/perspectives/hipaa-encryption-protocols-2025-updates)

**AWS Documentation:**

- [RDS Encryption Best Practices](https://docs.aws.amazon.com/prescriptive-guidance/latest/encryption-best-practices/rds.html)
- [AWS KMS Best Practices](https://docs.aws.amazon.com/kms/latest/developerguide/best-practices.html)
- [Architecting for HIPAA on AWS](https://docs.aws.amazon.com/whitepapers/latest/architecting-hipaa-security-and-compliance-on-aws/amazon-rds-for-postgresql.html)

**Standards:**

- [NIST FIPS 140-2](https://csrc.nist.gov/publications/detail/fips/140/2/final)
- [TLS 1.2/1.3 Specifications](https://datatracker.ietf.org/doc/html/rfc5246)

---

## Next Steps

Continue to the next section:

**[→ Audit Logging](03-audit-logging.md)**

Or explore related topics:

- [Authentication](01-authentication.md) - Protect encrypted data with auth
- [Data Handling](04-data-handling.md) - Manage encrypted data lifecycle

---

_Last Updated: November 2025_
_Part of the HIPAA Compliance Handbook - Backend Playbook_
