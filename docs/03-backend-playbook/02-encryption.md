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

### PostgreSQL Database Encryption

**DevOps Checklist:**

- ✅ Database storage encryption at rest (AES-256)
- ✅ SSL/TLS enforced for all connections
- ✅ Backups encrypted with same key
- ✅ Database in private network (no public access)

**What to verify:**
- Encryption enabled at creation (cannot add later)
- Customer-managed keys with rotation enabled
- Connection strings require SSL/TLS

#### PostgreSQL Force SSL Configuration

```sql
-- Force SSL for all connections (HIPAA REQUIRED)
ssl = on
ssl_min_protocol_version = 'TLSv1.2'
```

#### Connection String with SSL

```typescript
// Prisma: sslmode=require or verify-full
DATABASE_URL = "postgresql://user:pass@host:5432/db?sslmode=require"

// TypeORM: ssl: { rejectUnauthorized: true }
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
// ❌ Bad: Column-level encryption complicates queries
model Patient {
  ssnEncrypted String  // Cannot search/index!
}

// ✅ Good: Database-level encryption is sufficient
// Full-disk encryption protects all data
// Application sees plaintext, database stores encrypted
```

**⚠️ Important:** We determined column-level encryption was **NOT necessary** for our HIPAA project. RDS encryption with KMS was sufficient.

---

## Data in Transit Encryption

### TLS/HTTPS Requirements

All communication containing ePHI must use **TLS 1.2 or higher**:

**DevOps Checklist:**

- ✅ TLS 1.2+ enforced on load balancer/API gateway
- ✅ Valid SSL certificate configured
- ✅ HTTP redirects to HTTPS (no plaintext allowed)
- ✅ HSTS headers enabled

**What to verify:**
- SSL policy set to TLS 1.2+ minimum
- Certificate auto-renewal configured
- No self-signed certs in production

#### NestJS HTTPS Enforcement

```typescript
// Install helmet for security headers
app.use(helmet({ hsts: { maxAge: 31536000 } }))

// ✅ Good: HSTS header forces HTTPS
// ✅ Good: upgradeInsecureRequests in CSP
```

#### API Client Configuration (Frontend)

```typescript
// ✅ Good: Reject HTTP requests
if (url.startsWith("http://")) {
  throw new Error("HTTPS required")
}

// ✅ Good: Enforce TLS 1.2+
httpsAgent: new https.Agent({ minVersion: "TLSv1.2" })
```

---

## Key Management

### Key Management Best Practices

**DevOps Checklist:**

- ✅ Use customer-managed encryption keys (not provider defaults)
- ✅ Automatic key rotation enabled (annual minimum)
- ✅ Least-privilege access to keys
- ✅ All key usage logged and monitored
- ✅ Keys protected from deletion (retention policy)

**What to verify:**
- Key rotation is enabled and working
- Only authorized services/roles can use keys
- Failed key access attempts trigger alerts
- Keys cannot be accidentally deleted

---

## Backup Encryption

**DevOps Checklist:**

- ✅ Automated backups encrypted (inherit from database)
- ✅ Manual snapshots encrypted with same key
- ✅ Object storage backups encrypted (S3/similar)
- ✅ HTTPS required for backup uploads
- ✅ Backups in private storage (no public access)
- ✅ Retention: 30 days recent + long-term archive (6+ years)

**What to verify:**
- All backups are encrypted before leaving server
- Backup storage has versioning enabled
- Archive strategy transitions old backups to cold storage
- Backups are tested regularly for restore

---

## Testing Encryption

```typescript
// ✅ Test: Database encryption enabled
expect(dbInstance.storageEncrypted).toBe(true)

// ✅ Test: SSL required for connections
await expect(connectWithoutSSL()).rejects.toThrow()

// ✅ Test: HTTP rejected
await expect(axios.get("http://api")).rejects.toThrow()

// ✅ Test: TLS 1.2+ enforced
const response = await axios.get(url, {
  httpsAgent: { minVersion: "TLSv1.2" }
})
expect(response.status).toBe(200)
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
