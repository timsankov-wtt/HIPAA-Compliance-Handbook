# Backend Playbook

> Comprehensive guide for backend developers building HIPAA-compliant systems with NestJS and PostgreSQL

---

## Overview

As a backend developer, you are the **first line of defense** for protecting Protected Health Information (PHI). Your code, architecture decisions, and database configurations directly impact whether your application meets HIPAA requirements.

This playbook provides practical guidance for implementing HIPAA technical safeguards in backend systems, with specific examples for **NestJS**, **PostgreSQL**, and **AWS** infrastructure.

---

## Backend Team Responsibilities

When building HIPAA-compliant backend systems, you are responsible for:

### Data Protection
- Implementing encryption for all PHI at rest and in transit
- Ensuring secure database configurations
- Managing encryption keys properly

### Access Control
- Authenticating users securely
- Enforcing role-based access control (RBAC)
- Implementing multi-factor authentication (MFA)
- Managing session lifecycles

### Audit & Monitoring
- Logging all PHI access events
- Creating immutable audit trails
- Ensuring logs don't contain PHI themselves
- Implementing real-time security monitoring

### Data Lifecycle
- Properly handling PHI throughout its lifecycle
- Implementing secure deletion (no soft deletes!)
- Managing data retention policies
- Ensuring backup security

---

## Technical Safeguards Overview

HIPAA Security Rule requires three types of safeguards. Backend teams primarily focus on **Technical Safeguards**:

### 1. Access Control (§164.312(a)(1))
- **Unique User Identification** - Each user has a unique identifier
- **Emergency Access** - Procedures for accessing ePHI during emergencies
- **Automatic Logoff** - Sessions terminate after inactivity
- **Encryption & Decryption** - Mechanisms to encrypt/decrypt ePHI

### 2. Audit Controls (§164.312(b))
- Record and examine activity in systems containing ePHI
- Who accessed what, when, and from where

### 3. Integrity (§164.312(c))
- Ensure ePHI is not improperly altered or destroyed
- Mechanisms to corroborate that ePHI hasn't been tampered with

### 4. Person or Entity Authentication (§164.312(d))
- Verify that a person or entity is who they claim to be
- MFA now required for remote access (2025 update)

### 5. Transmission Security (§164.312(e))
- Protect ePHI transmitted over electronic networks
- Encryption mandatory (2025 update)

---

## Architecture Patterns for HIPAA Compliance

### Defense in Depth

Never rely on a single security mechanism. Layer multiple protections:

```
User Request
    ↓
[1. API Gateway - TLS/HTTPS]
    ↓
[2. Authentication - JWT/MFA]
    ↓
[3. Authorization - RBAC Guards]
    ↓
[4. Audit Logging - Record Access]
    ↓
[5. Database - Encrypted at Rest]
    ↓
[6. Audit Logging - Record Result]
    ↓
Response
```

### Least Privilege Access

Grant only the minimum permissions necessary:

- Database users should have limited permissions
- Application services use dedicated IAM roles
- Users only access data they need for their job function
- Regular review and revocation of unused permissions

### Fail Secure

When something goes wrong, default to denying access:

```javascript
// ✅ Good: Explicit permission check
if (user.hasPermission('PHI_READ') && user.canAccess(patientId)) {
  return patientData;
}
throw new ForbiddenException(); // Fail secure

// ❌ Bad: Implicit trust
if (!user.isBanned) {
  return patientData; // Could expose data if logic has bug
}
```

### Audit Everything

Every PHI access event must be logged:

- **Who** - User ID, role
- **What** - Action performed (read, write, delete)
- **When** - Timestamp (UTC)
- **Where** - IP address, system
- **Resource** - What PHI was accessed (patient ID, record type)
- **Result** - Success or failure

---

## NestJS + PostgreSQL Architecture

Our recommended architecture for HIPAA-compliant backends:

```
┌─────────────────────────────────────────────┐
│  API Layer (NestJS)                         │
│  - Controllers (input validation)           │
│  - Guards (authentication/authorization)    │
│  - Interceptors (audit logging)             │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│  Business Logic Layer                       │
│  - Services (application logic)             │
│  - DTOs (data transfer objects)             │
│  - Validation pipes                         │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│  Data Access Layer                          │
│  - Prisma ORM (type-safe queries)           │
│  - Repository pattern                       │
│  - Query parameterization (SQL injection)   │
└─────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────┐
│  Database (PostgreSQL)                      │
│  - Encryption at rest (AWS KMS)             │
│  - SSL/TLS connections                      │
│  - Row-level security                       │
│  - Audit logging (pgAudit)                  │
└─────────────────────────────────────────────┘
```

---

## Key Principles

### 1. Never Trust Input
Always validate and sanitize all input, even from authenticated users.

### 2. Encrypt Everything
All PHI must be encrypted at rest and in transit. No exceptions (2025 requirement).

### 3. Log All Access
Every time PHI is accessed, create an audit log entry. But never log PHI itself!

### 4. Implement RBAC
Users should only see data they're authorized to access based on their role.

### 5. Fail Securely
When errors occur, don't expose sensitive information. Log details securely, show generic errors to users.

### 6. Use Parameterized Queries
Always use Prisma/ORM or parameterized queries. Never concatenate user input into SQL.

### 7. Secure Session Management
Implement proper session timeouts, secure token storage, and automatic logoff.

### 8. Hard Delete Only
No soft deletes for patient data - HIPAA requires actual deletion when requested.

---

## Playbook Contents

This playbook is organized into focused sections:

### [1. Authentication & Authorization](01-authentication.md)
- JWT strategies and MFA implementation
- Role-Based Access Control (RBAC)
- Password policies and session management
- NestJS Guards patterns

### [2. Encryption](02-encryption.md)
- Data at rest encryption (PostgreSQL/RDS)
- Data in transit encryption (TLS 1.2+)
- AWS KMS integration and key management
- Why column-level encryption is often unnecessary

### [3. Audit Logging](03-audit-logging.md)
- What to log and what NOT to log
- Structured logging with NestJS
- CloudWatch Logs integration
- Log retention and integrity

### [4. Data Handling](04-data-handling.md)
- Data minimization principle
- Hard delete requirements (no soft deletes!)
- Data retention and backup policies
- PostgreSQL best practices

### [5. Backend Checklist](05-checklist.md)
- Complete pre-deployment checklist
- Critical, Important, and Recommended items
- Validation criteria for each requirement

---

## Quick Reference: Top 10 Backend Requirements

Before deploying any backend code that handles PHI:

- [ ] **All API endpoints use HTTPS/TLS 1.2+** - No HTTP allowed
- [ ] **MFA enabled for remote access** - Required as of 2025
- [ ] **All database connections encrypted** - PostgreSQL SSL mode required
- [ ] **Audit logging on all PHI access** - Who, what, when, where
- [ ] **RBAC implemented and tested** - Users only see authorized data
- [ ] **No PHI in application logs** - Use record IDs, not actual PHI
- [ ] **Parameterized queries only** - Prevent SQL injection
- [ ] **Session timeouts configured** - Auto-logout after inactivity
- [ ] **Error messages don't expose PHI** - Generic errors to users
- [ ] **Hard delete implemented** - No soft deletes for patient data

---

## Technology Stack

This playbook provides specific guidance for:

**Backend Framework:**
- NestJS (TypeScript)
- Express.js underlying

**Database:**
- PostgreSQL (AWS RDS)
- Prisma ORM

**Authentication:**
- JWT (JSON Web Tokens)
- Passport.js strategies
- AWS Cognito (optional)

**Logging:**
- Winston or Pino (structured logging)
- AWS CloudWatch Logs

**Encryption:**
- AWS KMS (key management)
- TLS 1.2+ (in transit)
- AES-256 (at rest)

---

## Common Backend Vulnerabilities

Be especially vigilant about these common security issues:

### SQL Injection
**Risk:** Attackers manipulate queries to access unauthorized data
**Prevention:** Always use Prisma ORM or parameterized queries

### Broken Authentication
**Risk:** Weak passwords, missing MFA, poor session management
**Prevention:** Implement strong auth, MFA, and session timeouts

### Sensitive Data Exposure
**Risk:** PHI in logs, error messages, or unencrypted storage
**Prevention:** Sanitize logs, encrypt everything, generic error messages

### Broken Access Control
**Risk:** Users accessing data they shouldn't
**Prevention:** Implement RBAC, test authorization thoroughly

### Security Misconfiguration
**Risk:** Default passwords, unnecessary services, verbose errors
**Prevention:** Security hardening, least privilege, configuration reviews

### Insufficient Logging & Monitoring
**Risk:** Breaches go undetected for months
**Prevention:** Comprehensive audit logging, real-time monitoring

---

## Development Workflow

### 1. Before Writing Code
- Review this playbook's relevant sections
- Understand what data is PHI
- Plan your audit logging strategy
- Design RBAC model

### 2. During Development
- Use TypeScript for type safety
- Write security-focused unit tests
- Test authorization edge cases
- Never commit secrets to git

### 3. Before Deployment
- Complete [Backend Checklist](05-checklist.md)
- Code review with security focus
- Test with production-like data (de-identified)
- Verify audit logs are working

### 4. After Deployment
- Monitor audit logs daily
- Review access patterns weekly
- Update dependencies monthly
- Conduct security audits quarterly

---

## Getting Help

**Questions about implementation?**
- Refer to specific sections in this playbook
- Check [Code Examples](../06-appendices/code-examples.md)
- Review [AWS Services Guide](../06-appendices/aws-services.md)

**Need to evaluate a third-party service?**
- See [Third-Party Vendors](../02-getting-started/04-third-party-vendors.md)
- Review [BAA Requirements](../02-getting-started/02-baa-requirements.md)

**Security concerns?**
- Consult your Privacy/Security Officer
- Review [Risk Assessment](../02-getting-started/03-risk-assessment.md)

---

## Next Steps

Start with the first section and work through each topic:

1. **[Authentication & Authorization →](01-authentication.md)**

Or jump to a specific topic:
- [Encryption](02-encryption.md)
- [Audit Logging](03-audit-logging.md)
- [Data Handling](04-data-handling.md)
- [Backend Checklist](05-checklist.md)

---

*Last Updated: November 2025*
*Part of the HIPAA Compliance Handbook*
