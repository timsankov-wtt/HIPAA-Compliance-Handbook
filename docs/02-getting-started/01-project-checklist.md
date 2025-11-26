# Project Checklist

> A practical guide for starting a HIPAA-compliant project from day one

---

## Do I Need HIPAA Compliance?

Use this decision tree to determine if your project requires HIPAA compliance:

```
START: Does your project handle any health information?
│
├─ NO → HIPAA likely doesn't apply (but verify with legal counsel)
│
└─ YES → Will you store, process, or transmit:
    │      - Patient names with health conditions?
    │      - Medical records or diagnoses?
    │      - Appointment schedules with patient identifiers?
    │      - Billing information related to healthcare?
    │      - Lab results or prescriptions?
    │
    ├─ NO → Are you handling ONLY de-identified data?
    │   │
    │   ├─ YES (properly de-identified) → HIPAA may not apply
    │   │                                  (verify de-identification method)
    │   │
    │   └─ NO or UNSURE → Continue below
    │
    └─ YES → Are you:
        │     - A healthcare provider?
        │     - A health insurance company?
        │     - Working FOR a healthcare organization?
        │     - Building software for healthcare?
        │     - Hosting/storing healthcare data?
        │
        └─ YES → ✅ HIPAA COMPLIANCE REQUIRED
                  You are either a Covered Entity or Business Associate
```

**Common Scenarios:**

| Scenario | HIPAA Required? |
|----------|----------------|
| Building patient portal for hospital | ✅ YES - Business Associate |
| EHR/EMR system development | ✅ YES - Business Associate |
| Telemedicine platform with patient records | ✅ YES - Covered Entity or BA |
| Healthcare appointment scheduling app | ✅ YES - Business Associate |
| Medical billing software | ✅ YES - Business Associate |
| Fitness app (no medical records) | ❌ NO - Unless integrating with health data |
| Anonymous health research (properly de-identified) | ❌ NO - If truly de-identified |
| General wellness app (no PHI) | ❌ NO - But be careful about data collection |

**⚠️ When in Doubt:** If you handle ANY information that could identify a patient combined with health information, assume HIPAA applies and consult with legal counsel.

---

## Pre-Project Checklist

Before writing a single line of code, complete these essential planning steps:

### 📋 Planning Phase

#### Legal & Compliance Foundation

- [ ] **Determine HIPAA applicability** - Use decision tree above
- [ ] **Identify your role** - Are you a Covered Entity or Business Associate?
- [ ] **Consult legal counsel** - Verify compliance requirements for your specific use case
- [ ] **Document compliance scope** - What PHI will you handle? Where? How?
- [ ] **Budget for compliance** - Factor in additional infrastructure, tools, and time

#### Team & Responsibilities

- [ ] **Assign Privacy Officer** - Person responsible for privacy policies and procedures
- [ ] **Assign Security Officer** - Person responsible for technical security measures
  - *Note: Same person can serve both roles in small teams*
- [ ] **Define team roles** - Who handles PHI? Who has access? Who audits?
- [ ] **Plan HIPAA training** - All team members need compliance training
- [ ] **Create incident response team** - Who responds to breaches or security events?

#### Risk Assessment & Documentation

- [ ] **Conduct initial risk assessment** - Identify potential vulnerabilities (see [Risk Assessment Guide](03-risk-assessment.md))
- [ ] **Document data flows** - Map where PHI enters, moves through, and exits your system
- [ ] **Identify all PHI touchpoints** - Databases, logs, backups, third-party services
- [ ] **Create security policies document** - Written policies for data handling
- [ ] **Document retention policies** - How long will you keep PHI? (typically 6+ years)

#### Third-Party Vendor Assessment

- [ ] **List ALL third-party services** - Cloud providers, analytics, email, SMS, monitoring tools
- [ ] **Verify HIPAA eligibility** - Check if vendors offer BAAs
- [ ] **Obtain BAA quotes** - Some vendors charge premium for HIPAA compliance
- [ ] **Sign BAAs BEFORE development** - Never handle PHI without BAAs in place
- [ ] **Document vendor compliance status** - Maintain registry of all BAAs

**Key Vendors to Evaluate:**
- Cloud infrastructure (AWS, Azure, GCP)
- Email services (check HIPAA-compliant options)
- SMS/notification services
- Error tracking (Sentry, Rollbar)
- Analytics platforms
- Payment processors
- AI/ML services (like ElevenLabs for voice)

📖 **Detailed Guide:** [BAA Requirements](02-baa-requirements.md) | [Third-Party Vendors](04-third-party-vendors.md)

---

### 💻 Development Phase

#### Infrastructure Setup

- [ ] **Sign AWS BAA via Artifact** - Do this FIRST (see [AWS Setup Guide](../05-devops-playbook/01-aws-setup.md))
- [ ] **Use only HIPAA-eligible services** - Verify each AWS service is on approved list
- [ ] **Enable encryption at rest** - All databases, file storage, backups
- [ ] **Enable encryption in transit** - TLS 1.2+ for all data transmission
- [ ] **Configure network segmentation** - Isolate PHI-containing systems
- [ ] **Set up VPC with private subnets** - Database and sensitive services in private networks
- [ ] **Implement IAM least privilege** - Minimal permissions for all users and services

#### Authentication & Access Control

- [ ] **Implement strong authentication** - JWT, OAuth, or similar secure method
- [ ] **Enable Multi-Factor Authentication (MFA)** - Required for all remote access (2025 mandate)
- [ ] **Implement Role-Based Access Control (RBAC)** - Define roles: admin, provider, patient, etc.
- [ ] **Set session timeouts** - Auto-logout after inactivity (typically 15-30 minutes)
- [ ] **Create password policies** - Minimum complexity, rotation requirements
- [ ] **Implement account lockout** - After failed login attempts

#### Audit Logging

- [ ] **Log ALL PHI access** - Every read, write, update, delete operation
- [ ] **Implement structured logging** - JSON format with required fields (who, what, when, where)
- [ ] **Set up CloudWatch Logs** - Centralized logging for all services
- [ ] **Configure log retention** - Minimum 6 years for audit logs
- [ ] **Encrypt audit logs** - Logs must be encrypted and tamper-proof
- [ ] **Never log PHI content** - Log that data was accessed, not the data itself

#### Database & Data Security

- [ ] **Enable database encryption** - RDS encryption with AWS KMS
- [ ] **Use parameterized queries** - Prevent SQL injection
- [ ] **Implement data validation** - Server-side validation for all inputs
- [ ] **Plan data retention strategy** - Automated cleanup after retention period
- [ ] **Use hard deletes (not soft deletes)** - HIPAA requires actual deletion when requested
- [ ] **Set up automated backups** - Daily backups with encryption
- [ ] **Test backup restoration** - Verify backups work before production

#### Application Security

- [ ] **Implement input validation** - Both client and server-side
- [ ] **Prevent XSS attacks** - Sanitize all user inputs and outputs
- [ ] **Use Content Security Policy (CSP)** - Prevent script injection
- [ ] **Implement CORS properly** - Restrict API access to authorized domains
- [ ] **Never expose PHI in URLs** - No PHI in query parameters or paths
- [ ] **Secure error handling** - No PHI in error messages or stack traces
- [ ] **Remove debug code** - No console.log() or debug outputs in production

---

### 🚀 Deployment Phase

#### Pre-Deployment Security

- [ ] **Complete security checklist** - Backend, Frontend, DevOps checklists
- [ ] **Conduct penetration testing** - Annual requirement, do before launch
- [ ] **Perform vulnerability scan** - Use automated tools (required every 6 months)
- [ ] **Review all IAM permissions** - Ensure least privilege everywhere
- [ ] **Verify encryption status** - All data at rest and in transit encrypted
- [ ] **Test MFA implementation** - Ensure all remote access requires MFA

#### Monitoring & Alerting

- [ ] **Set up CloudWatch alarms** - For failed logins, unusual access patterns
- [ ] **Enable AWS GuardDuty** - Threat detection for AWS environment
- [ ] **Configure AWS Config** - Compliance monitoring and drift detection
- [ ] **Set up CloudTrail** - Log all AWS API calls
- [ ] **Create alert notifications** - SNS/email for security events
- [ ] **Test incident response** - Run through breach notification procedure

#### Documentation

- [ ] **Document system architecture** - Network diagrams, data flows
- [ ] **Create runbooks** - Incident response, backup restoration, disaster recovery
- [ ] **Write privacy policies** - Patient-facing privacy notices
- [ ] **Document security procedures** - For audit purposes
- [ ] **Create training materials** - For team onboarding

#### Compliance Verification

- [ ] **Review Business Associate Agreements** - All vendors signed before launch
- [ ] **Conduct final risk assessment** - Document any remaining risks
- [ ] **Create compliance documentation package** - Policies, procedures, risk assessments
- [ ] **Set annual audit schedule** - Risk assessments, penetration tests, vulnerability scans
- [ ] **Establish breach notification process** - Know the 60-day timeline

---

### 🔄 Maintenance Phase (Post-Launch)

#### Ongoing Compliance

- [ ] **Conduct annual risk assessments** - Required as of 2025 updates
- [ ] **Perform vulnerability scans** - Every 6 months (2025 requirement)
- [ ] **Run penetration tests** - Annually minimum
- [ ] **Review and update policies** - Quarterly or when systems change
- [ ] **Maintain audit log access** - Regular review of access logs
- [ ] **Test disaster recovery plan** - Quarterly backup restoration tests

#### Team Management

- [ ] **Conduct HIPAA training** - All new employees before PHI access
- [ ] **Annual refresher training** - For all team members
- [ ] **Review access permissions** - Quarterly review of who has access to what
- [ ] **Offboarding procedures** - Immediately revoke access when employees leave
- [ ] **Incident response drills** - Practice breach notification procedures

#### Vendor Management

- [ ] **Maintain BAA registry** - Track all vendor agreements and renewal dates
- [ ] **Review vendor compliance** - Annual review of third-party security
- [ ] **Evaluate new services** - HIPAA assessment before adding any tool
- [ ] **Monitor vendor breaches** - Stay informed of incidents affecting your vendors

#### Monitoring & Improvement

- [ ] **Review CloudWatch metrics** - Weekly check of access patterns
- [ ] **Analyze audit logs** - Look for anomalies or suspicious activity
- [ ] **Update security patches** - Regular OS and dependency updates
- [ ] **Monitor compliance news** - Stay current with HIPAA updates
- [ ] **Continuous improvement** - Regular security enhancements

---

## Team Roles & Responsibilities

### Privacy Officer

**Responsibilities:**
- Develop and maintain privacy policies
- Ensure patient rights are protected (access, amendment, accounting)
- Oversee privacy training programs
- Investigate privacy complaints
- Maintain Notice of Privacy Practices (NPP)
- Coordinate with legal counsel on privacy matters

**Technical Skills Needed:**
- Understanding of HIPAA Privacy Rule
- Policy documentation
- Risk assessment
- Incident investigation

**Time Commitment:** 10-20% of time for small teams, full-time for larger organizations

---

### Security Officer

**Responsibilities:**
- Develop and maintain security policies
- Conduct risk assessments and security audits
- Oversee implementation of technical safeguards
- Monitor security incidents and breaches
- Manage access controls and authentication
- Coordinate with DevOps on infrastructure security

**Technical Skills Needed:**
- Understanding of HIPAA Security Rule
- Infrastructure security (AWS, networking)
- Encryption and key management
- Incident response
- Security monitoring and logging

**Time Commitment:** 20-40% of time for small teams, full-time for larger organizations

---

### Developer Responsibilities

**All Developers Must:**
- Complete HIPAA training before accessing any systems
- Follow secure coding practices
- Never log PHI in application logs
- Implement proper input validation and sanitization
- Use only approved third-party services
- Report security concerns immediately
- Participate in code reviews with security focus

**Backend Developers:**
- Implement authentication and authorization
- Set up audit logging for all PHI access
- Configure database encryption
- Develop secure API endpoints
- Implement data retention and deletion procedures

**Frontend Developers:**
- Implement secure data transmission
- Prevent XSS and injection attacks
- Handle authentication tokens securely
- Never store PHI in browser storage
- Implement session timeouts

---

### DevOps Responsibilities

**Infrastructure:**
- Configure HIPAA-compliant AWS environment
- Enable encryption at rest and in transit
- Implement network segmentation
- Set up monitoring and alerting
- Manage IAM roles and policies

**Operations:**
- Maintain backup and disaster recovery
- Perform regular security updates
- Monitor system logs for anomalies
- Conduct vulnerability scans
- Manage SSL/TLS certificates

**Compliance:**
- Maintain CloudTrail and CloudWatch logs
- Configure AWS Config for compliance monitoring
- Coordinate penetration testing
- Document infrastructure architecture

---

## Estimated Timeline

### Small Team (3-5 developers)

| Phase | Duration | Key Activities |
|-------|----------|----------------|
| **Planning** | 2-3 weeks | Risk assessment, vendor evaluation, architecture design |
| **Infrastructure Setup** | 1-2 weeks | AWS account, BAAs, network configuration, encryption |
| **Core Development** | 8-12 weeks | Application development with security controls |
| **Security Implementation** | 3-4 weeks | Audit logging, MFA, RBAC, testing |
| **Testing & Audit** | 2-3 weeks | Penetration testing, vulnerability scans, compliance review |
| **Documentation** | 1-2 weeks | Policies, procedures, training materials |
| **Total** | **17-26 weeks** | (4-6 months) |

**Note:** This assumes team members have some HIPAA knowledge. Add 2-4 weeks for training if starting from scratch.

### Medium Team (6-15 developers)

| Phase | Duration |
|-------|----------|
| Planning | 3-4 weeks |
| Setup & Development | 12-16 weeks |
| Security & Testing | 4-6 weeks |
| Documentation & Training | 2-3 weeks |
| **Total** | **21-29 weeks** (5-7 months) |

### Large Team (15+ developers)

- Add coordination overhead
- More complex architecture
- Additional security reviews
- **Typical Timeline:** 6-9 months

---

## Common Pitfalls in Project Planning

### ❌ Starting Development Before BAAs

**Problem:** Building features that store PHI before signing vendor agreements
**Impact:** Must rebuild infrastructure or face non-compliance
**Solution:** Sign ALL BAAs before writing code that touches PHI

### ❌ Underestimating Compliance Time

**Problem:** "We'll add security later"
**Impact:** Massive refactoring, delayed launch, technical debt
**Solution:** Build security in from day one, not as afterthought

### ❌ Ignoring Third-Party Costs

**Problem:** Discovering HIPAA-compliant services cost 5-10x more
**Impact:** Budget overruns, vendor switching mid-project
**Solution:** Get quotes for ALL services during planning phase

**Examples:**
- Twilio HIPAA: $10,000+ setup fee
- SendGrid: NOT HIPAA-compliant
- Standard monitoring vs HIPAA monitoring: Often 3-5x cost difference

### ❌ Inadequate Logging

**Problem:** Adding audit logging as afterthought
**Impact:** Difficult to retrofit, incomplete audit trails
**Solution:** Implement logging infrastructure before feature development

### ❌ Assuming Cloud Provider Handles Everything

**Problem:** "AWS is HIPAA-compliant, so we are too"
**Impact:** Misconfigured services, compliance violations
**Solution:** Understand Shared Responsibility Model - YOU configure services securely

### ❌ No Testing Plan

**Problem:** Skipping penetration testing and vulnerability scans
**Impact:** Production launch with security holes, failed audits
**Solution:** Budget for external security testing (required annually)

### ❌ Poor Documentation

**Problem:** No written policies or procedures
**Impact:** Failed audits, inability to prove compliance
**Solution:** Document everything - policies, configurations, decisions

### ❌ Soft Delete Patterns

**Problem:** Using soft deletes (marking records as deleted)
**Impact:** Violates HIPAA right to deletion
**Solution:** Implement hard deletion with audit trail

### ❌ Insufficient Training

**Problem:** Developers unaware of HIPAA requirements
**Impact:** PHI in logs, insecure code, compliance violations
**Solution:** Mandatory training before any PHI access

---

## Success Criteria

Your project planning is complete when you can answer "YES" to all:

- [ ] We know exactly what PHI we will handle
- [ ] All third-party vendors have signed BAAs
- [ ] Budget includes compliance infrastructure costs
- [ ] Team roles are assigned (Privacy Officer, Security Officer)
- [ ] Initial risk assessment is documented
- [ ] Infrastructure architecture is designed with security first
- [ ] Development team has completed HIPAA training
- [ ] Audit logging strategy is defined
- [ ] Backup and disaster recovery plan exists
- [ ] Incident response procedures are documented
- [ ] Timeline accounts for security testing and audits

---

## What's Next?

Now that you have your project checklist:

1. **[BAA Requirements](02-baa-requirements.md)** - Understand and obtain Business Associate Agreements
2. **[Risk Assessment](03-risk-assessment.md)** - Conduct your first risk assessment
3. **[Third-Party Vendors](04-third-party-vendors.md)** - Evaluate and select HIPAA-compliant services
4. **[Backend Playbook](../03-backend-playbook/README.md)** - Start implementing technical safeguards

---

## Additional Resources

**Planning Tools:**
- [HHS Security Risk Assessment Tool](https://www.healthit.gov/topic/privacy-security-and-hipaa/security-risk-assessment-tool)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)

**Templates:**
- Sample Risk Assessment: See [Risk Assessment Guide](03-risk-assessment.md)
- Sample BAA: [HHS Model BAA](https://www.hhs.gov/hipaa/for-professionals/covered-entities/sample-business-associate-agreement-provisions/index.html)

---

*Last Updated: November 2025*
*Next: [BAA Requirements →](02-baa-requirements.md)*
