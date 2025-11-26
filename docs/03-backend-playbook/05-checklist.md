---
layout: default
title: Backend Compliance Checklist
parent: Backend Playbook
nav_order: 6
---

# Backend Compliance Checklist

> Before-deployment checklist for backend services handling PHI.
>
> Use this document together with:
>
> - [Authentication & Authorization](01-authentication.md)
> - [Encryption](02-encryption.md)
> - [Audit Logging](03-audit-logging.md)
> - [Data Handling](04-data-handling.md)
> - [DevOps Infrastructure](../05-devops-playbook/02-infrastructure.md)
> - [Backup & DR](../05-devops-playbook/04-backup-dr.md)

---

## How to Use This Checklist

- Run this checklist **before every production deployment** that touches systems with PHI.
- Each item is marked as:
  - **Critical (Must Have)** – required for HIPAA alignment.
  - **Important (Should Have)** – strongly recommended; plan remediation if missing.
  - **Recommended (Nice to Have)** – improves security posture and operability.
- Every item includes **validation criteria** and link(s) to relevant handbook sections.

For each unchecked item, either:

- Implement it before go-live, or
- Document an explicit risk acceptance and remediation plan.

---

## Critical (Must Have)

### Authentication & Authorization

- [ ] **MFA enforced for all remote access to systems with PHI**  
       **Validate:** Attempt login without second factor from an external network – access must be denied. Verify MFA policy covers admins, support staff, and any VPN/bastion access.  
       **See:** [Authentication & Authorization](01-authentication.md)

- [ ] **All PHI endpoints protected by RBAC-based NestJS guards**  
       **Validate:** Review NestJS routes/controllers – no PHI-related route is marked public/unguarded. Unit or e2e tests confirm unauthorized roles receive `403 Forbidden`.  
       **See:** [Authentication & Authorization](01-authentication.md)

- [ ] **Session and token lifetimes configured with secure timeouts**  
       **Validate:** Inspect token TTLs and session settings; inactive sessions expire according to policy, refresh tokens are bounded, and idle browser sessions are invalidated.  
       **See:** [Authentication & Authorization](01-authentication.md)

### Encryption

- [ ] **All PHI stored only on encrypted services (RDS, S3, backups) with AWS KMS**  
       **Validate:** Confirm RDS and any S3 buckets containing PHI have encryption-at-rest enabled with KMS keys; no PHI on unencrypted EBS volumes or local disks.  
       **See:** [Encryption](02-encryption.md), [DevOps Infrastructure](../05-devops-playbook/02-infrastructure.md)

- [ ] **TLS enforced end-to-end (HTTPS only, TLS 1.2+)**  
       **Validate:** HTTP requests are redirected/rejected; SSL policy on ALB/API Gateway set to TLS 1.2+; no plaintext internal hops for PHI traffic.  
       **See:** [Encryption](02-encryption.md), [DevOps Infrastructure](../05-devops-playbook/02-infrastructure.md)

- [ ] **Secrets and encryption keys never stored in code or plain env files**  
       **Validate:** Search repo and CI/CD configs for hardcoded secrets; verify use of AWS Secrets Manager/SSM Parameter Store and KMS for all sensitive material.  
       **See:** [Encryption](02-encryption.md), [AWS Setup](../05-devops-playbook/01-aws-setup.md)

### Audit Logging

- [ ] **Audit logging implemented for all PHI access (read/write/delete)**  
       **Validate:** For a sample PHI record, trigger read, update, and delete; verify corresponding audit entries exist with `who / what / when / where`.  
       **See:** [Audit Logging](03-audit-logging.md), [Data Handling](04-data-handling.md)

- [ ] **Audit logs stored centrally (e.g., CloudWatch Logs) with ≥6 years retention**  
       **Validate:** Check log groups and retention policies; confirm encryption enabled and that logs cannot be edited by application roles.  
       **See:** [Audit Logging](03-audit-logging.md), [Monitoring](../05-devops-playbook/03-monitoring.md)

- [ ] **Audit logs contain identifiers, not PHI content**  
       **Validate:** Inspect log samples – ensure no names, diagnoses, notes, or other PHI fields appear; only IDs, timestamps, and event metadata.  
       **See:** [Audit Logging](03-audit-logging.md)

### Data Handling

- [ ] **NO soft deletes for PHI – hard deletes with audit trail only**  
       **Validate:** Prisma models for PHI contain no `deletedAt` / `isDeleted` flags; deletion flows perform `delete` operations with a pre-delete audit log entry.  
       **See:** [Data Handling](04-data-handling.md)

- [ ] **Data minimization applied to models, queries, and DTOs**  
       **Validate:** API list endpoints and scheduled jobs do not select or return unnecessary PHI; DTOs for lists use initials or reduced fields where possible.  
       **See:** [Data Handling](04-data-handling.md)

- [ ] **Automated retention policy implemented for PHI with legal-hold support**  
       **Validate:** Scheduled job exists that deletes or archives data older than the configured retention; records under legal hold are excluded; deletions are logged.  
       **See:** [Data Handling](04-data-handling.md)

### API Security

- [ ] **All backend APIs are authenticated by default; explicit allowlist for non-PHI health/status endpoints**  
       **Validate:** Review global guards and route configuration; unauthenticated access is allowed only to strictly non-sensitive endpoints (e.g., `/health`).  
       **See:** [Authentication & Authorization](01-authentication.md)

- [ ] **Strong input validation and sanitization on all external inputs**  
       **Validate:** DTOs/validation pipes are in place for every request body/query param; fuzz testing or negative tests demonstrate rejection of malformed input.  
       **See:** [Authentication & Authorization](01-authentication.md), [Frontend Data Validation](../04-frontend-playbook/01-data-validation.md)

- [ ] **Rate limiting or abuse protection enabled for authentication and sensitive endpoints**  
       **Validate:** Confirm API gateway / load balancer / app middleware enforces rate limits and that limits are tested under load.  
       **See:** [Authentication & Authorization](01-authentication.md), [Infrastructure](../05-devops-playbook/02-infrastructure.md)

### Database Security

- [ ] **Database not publicly accessible; access restricted via VPC and security groups**  
       **Validate:** RDS instance in private subnets; security groups allow inbound traffic only from application layer and admin bastions, never from the public internet.  
       **See:** [Encryption](02-encryption.md), [Infrastructure](../05-devops-playbook/02-infrastructure.md)

- [ ] **Row-Level Security (RLS) enforced for multi-tenant PHI access**  
       **Validate:** RLS enabled on PHI tables; policies tested to ensure users only see allowed records; application sets `current_user_id` / similar context per request.  
       **See:** [Data Handling](04-data-handling.md)

- [ ] **Least-privilege database roles for application and operations**  
       **Validate:** Application uses a non-superuser role with minimal privileges; separate roles for migrations and read-only analytics where needed.  
       **See:** [Encryption](02-encryption.md), [Data Handling](04-data-handling.md)

---

## Important (Should Have)

### Authentication & Authorization

- [ ] **Account lockout and alerting after repeated failed login attempts**  
       **Validate:** Simulate multiple failed logins; account is temporarily locked and security team receives an alert or log entry.  
       **See:** [Authentication & Authorization](01-authentication.md), [Monitoring](../05-devops-playbook/03-monitoring.md)

- [ ] **Admin and support actions are auditable and scoped to dedicated roles**  
       **Validate:** Admin routes require elevated roles; sensitive support tools cannot be accessed with normal user accounts; all such actions are logged.  
       **See:** [Authentication & Authorization](01-authentication.md)

### Encryption

- [ ] **KMS key rotation enabled and documented for all PHI-related keys**  
       **Validate:** Check KMS key policies and rotation settings; rotation events recorded and, if necessary, tested in lower environments.  
       **See:** [Encryption](02-encryption.md)

- [ ] **Clients enforce certificate validation and do not ignore TLS errors**  
       **Validate:** Code/configs do not disable certificate validation; connection attempts with invalid certs fail as expected in tests.  
       **See:** [Encryption](02-encryption.md)

### Audit Logging

- [ ] **Standardized structured log format with correlation IDs**  
       **Validate:** Sample logs from different services share a common schema and correlation ID field; a single user action can be traced end-to-end.  
       **See:** [Audit Logging](03-audit-logging.md)

- [ ] **Predefined queries/dashboards for PHI access review**  
       **Validate:** CloudWatch Insights (or similar) queries exist to list PHI access by user/date/resource, and are reviewed periodically.  
       **See:** [Audit Logging](03-audit-logging.md), [Monitoring](../05-devops-playbook/03-monitoring.md)

### Data Handling

- [ ] **Clear separation of PHI vs non-PHI tables and services**  
       **Validate:** Schema review shows PHI concentrated in dedicated tables/models; non-PHI services do not depend on PHI tables.  
       **See:** [Data Handling](04-data-handling.md)

- [ ] **De-identified or aggregated datasets used for analytics where possible**  
       **Validate:** Analytics queries and exports avoid direct PHI; where identifiers are needed, use pseudonyms or tokens.  
       **See:** [Data Handling](04-data-handling.md)

### API Security

- [ ] **CORS configuration restricted to known, trusted frontend origins**  
       **Validate:** CORS settings list only production/staging domains; wildcards are not used for origins, headers, or methods when PHI is involved.  
       **See:** [Frontend Secure Transmission](../04-frontend-playbook/02-secure-transmission.md)

- [ ] **Error responses do not leak stack traces or sensitive configuration**  
       **Validate:** Force backend errors and inspect responses – users see generic messages; detailed errors go only to logs/monitoring systems.  
       **See:** [Audit Logging](03-audit-logging.md)

### Database Security

- [ ] **Regular PostgreSQL maintenance (VACUUM / ANALYZE) configured and monitored**  
       **Validate:** Auto-vacuum settings tuned; maintenance jobs and monitoring in place to prevent table bloat and performance degradation.  
       **See:** [Data Handling](04-data-handling.md)

- [ ] **Database configuration hardened for logging and access control**  
       **Validate:** Failed login attempts, connection counts, and slow queries are logged; default users/roles are reviewed and disabled where appropriate.  
       **See:** [Data Handling](04-data-handling.md), [Monitoring](../05-devops-playbook/03-monitoring.md)

---

## Recommended (Nice to Have)

### Authentication & Authorization

- [ ] **Use of managed authentication (e.g., AWS Cognito) where appropriate**  
       **Validate:** Where feasible, authentication is delegated to a managed, HIPAA-eligible service, reducing custom auth surface area.  
       **See:** [Authentication & Authorization](01-authentication.md)

### Encryption

- [ ] **Mutual TLS or service mesh for internal service-to-service communication**  
       **Validate:** Internal API calls between backend services are authenticated and encrypted using mTLS or a zero-trust service mesh.  
       **See:** [Encryption](02-encryption.md), [Infrastructure](../05-devops-playbook/02-infrastructure.md)

### Audit Logging

- [ ] **Anomaly detection on audit logs for unusual PHI access patterns**  
       **Validate:** Monitoring tooling flags abnormal access (e.g., bulk exports, off-hours access) and routes alerts to the security team.  
       **See:** [Audit Logging](03-audit-logging.md), [Monitoring](../05-devops-playbook/03-monitoring.md)

### Data Handling

- [ ] **Partitioning strategies for PHI-heavy tables to support retention and performance**  
       **Validate:** Large tables (e.g., audit logs, historical records) are partitioned by time or tenant; old partitions can be archived or dropped cleanly.  
       **See:** [Data Handling](04-data-handling.md)

### API Security

- [ ] **Automated security scanning (SAST/DAST) integrated into CI/CD for backend APIs**  
       **Validate:** CI/CD pipelines include static and/or dynamic security scans, and high-severity findings block deployment until resolved.  
       **See:** [Authentication & Authorization](01-authentication.md), [Infrastructure](../05-devops-playbook/02-infrastructure.md)

### Database Security

- [ ] **Separate clusters or schemas for PHI vs analytics workloads**  
       **Validate:** Analytics jobs operate on read replicas, de-identified exports, or separate schemas; production PHI database is not used directly for reporting by third-party tools.  
       **See:** [Data Handling](04-data-handling.md)

---

## Sign-off

Use this section to document final approval before go-live.

- **Backend Lead:** ****\*\*****\_****\*\*****  
  **Date:** ****\*\*****\_****\*\*****

- **Security/Privacy Officer:** ****\*\*****\_****\*\*****  
  **Date:** ****\*\*****\_****\*\*****

- **DevOps Lead:** ****\*\*****\_****\*\*****  
  **Date:** ****\*\*****\_****\*\*****

Notes / Known Deviations:

- ***
- ***
- ***
