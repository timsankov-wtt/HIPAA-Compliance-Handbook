# HIPAA Risk Assessment Guide

> Comprehensive guide to conducting HIPAA-required security risk assessments

---

## What is a HIPAA Risk Assessment?

A **HIPAA Security Risk Assessment** (also called Risk Analysis) is a thorough evaluation of potential risks and vulnerabilities to the confidentiality, integrity, and availability of electronic Protected Health Information (ePHI) held by your organization.

### Simple Definition

**A risk assessment answers these questions:**
- What ePHI do we have?
- Where is it stored, processed, or transmitted?
- What could go wrong?
- How likely is it to go wrong?
- What would be the impact if it did?
- How can we prevent or mitigate these risks?

### Legal Requirement

Under the HIPAA Security Rule, both covered entities and business associates **must conduct and document** a security risk assessment. It's not optional - it's one of the first requirements listed in the Security Rule (§164.308(a)(1)(ii)(A)).

### 2025 Updates: Now Annual

**Before 2025:** Risk assessments were required "periodically" (vague timeline)

**As of 2025:** The proposed HIPAA Security Rule updates mandate:
- ✅ **Formal risk assessments at least annually**
- ✅ **Vulnerability scans every 6 months**
- ✅ **Penetration testing at least annually**
- ✅ **Technology asset inventory maintenance**
- ✅ **Network mapping requirements**

**📖 Read more:** [2025 HIPAA Updates](../01-fundamentals/03-2025-updates.md)

---

## When to Conduct Risk Assessments

### Required Timing

**1. Initial Assessment (Before Launch)**
- Before handling any ePHI
- During system design and planning phase
- Identifies baseline risks to address

**2. Annual Assessment (2025 Requirement)**
- At least once per year
- Full comprehensive review
- Document changes since last assessment

**3. Significant Changes**
Conduct assessment after:
- New system implementations
- Major software upgrades
- Infrastructure changes (new cloud services, databases)
- Organizational changes (mergers, acquisitions)
- New third-party integrations
- Security incidents or breaches
- Significant policy changes

**4. Regulatory Triggers**
- OCR investigation or audit
- Breach notification requirements
- Compliance verification requests

### Recommended Frequency

**Minimum (Compliant):**
- Annual comprehensive assessment
- Vulnerability scans every 6 months
- Penetration testing annually

**Best Practice:**
- Quarterly lightweight reviews
- Continuous vulnerability scanning
- Ad-hoc assessments for changes
- Monthly threat monitoring

---

## Risk Assessment Process: Step-by-Step

### Phase 1: Preparation

#### Define Scope

**Determine What to Include:**
- [ ] All systems that create, receive, maintain, or transmit ePHI
- [ ] Physical locations (offices, data centers, remote work)
- [ ] Third-party services and vendors
- [ ] Mobile devices and endpoints
- [ ] Network infrastructure
- [ ] Backup and disaster recovery systems

**Document System Boundaries:**
```
Example Scope Statement:
"This risk assessment covers all electronic systems within [Company Name]
that handle ePHI, including:
- Production AWS environment (ECS, RDS, S3)
- Office network and workstations
- Mobile devices accessing patient data
- Third-party services with BAAs (AWS, Stripe)
- Backup systems and disaster recovery

Excluded from scope:
- Public marketing website (no PHI)
- Internal HR systems (no PHI)
- Development/staging environments (test data only)
"
```

#### Assemble Assessment Team

**Required Roles:**
- **Security Officer** - Leads assessment
- **Privacy Officer** - Ensures PHI handling compliance
- **IT/DevOps** - Technical infrastructure knowledge
- **Developers** - Application-level security
- **Management** - Resource allocation and decision authority
- **Optional: External Auditor** - Independent verification

**Time Commitment:**
- Small team (5-10 people): 40-80 hours total
- Medium team (10-50 people): 80-160 hours total
- Large organization: 160+ hours total

---

### Phase 2: Asset Inventory

Create comprehensive inventory of all technology assets that touch ePHI.

#### System Inventory Template

| System/Asset | Type | Location | ePHI Stored? | Sensitivity | Owner | Last Updated |
|--------------|------|----------|--------------|-------------|-------|--------------|
| Production DB | RDS PostgreSQL | AWS us-east-1 | Yes | Critical | DevOps | 2025-01-15 |
| API Server | ECS Container | AWS us-east-1 | Yes | Critical | Backend | 2025-01-15 |
| Web App | Next.js/React | CloudFront | No (transit only) | High | Frontend | 2025-01-10 |
| Backup Storage | S3 Bucket | AWS us-east-1 | Yes | Critical | DevOps | 2025-01-15 |
| Admin Laptops | MacBook Pro | Remote | Potential access | Medium | IT | 2024-12-01 |
| Office WiFi | Network | Office | No | Medium | IT | 2024-11-15 |

**Asset Categories to Document:**

**1. Infrastructure**
- Cloud services (AWS accounts, services)
- Servers and VMs
- Databases
- Storage systems (S3, file servers)
- Network devices (routers, firewalls, load balancers)
- Backup systems

**2. Applications**
- Patient-facing applications
- Administrative portals
- APIs and microservices
- Third-party integrations
- Mobile applications

**3. Endpoints**
- Employee workstations
- Mobile devices (phones, tablets)
- Remote access devices
- Medical devices (if applicable)

**4. Data Storage**
- Databases (production, staging)
- File storage
- Backups and archives
- Email systems
- Cloud storage

**Required Information for Each Asset:**
- [ ] Asset name and description
- [ ] Type/category
- [ ] Physical or logical location
- [ ] Does it contain ePHI? (Yes/No)
- [ ] ePHI data types (patient names, medical records, etc.)
- [ ] Sensitivity level (Critical/High/Medium/Low)
- [ ] System owner/responsible party
- [ ] Operating system and versions
- [ ] Software versions
- [ ] Network connections
- [ ] Authentication method
- [ ] Encryption status (at rest, in transit)
- [ ] Last update/patch date
- [ ] Backup status

---

### Phase 3: Threat Identification

Identify all reasonably anticipated threats to ePHI confidentiality, integrity, and availability.

#### Common Threat Categories

**1. Human Threats (Intentional)**
- External hackers and cybercriminals
- Malicious insiders (disgruntled employees)
- Social engineering attacks
- Phishing and spear-phishing
- Ransomware attacks
- Credential theft
- Unauthorized access attempts

**2. Human Threats (Unintentional)**
- Accidental data disclosure
- Misconfiguration of systems
- Lost or stolen devices
- Weak passwords
- Falling for phishing
- Improper disposal of data
- Unauthorized PHI access by employees

**3. Environmental Threats**
- Natural disasters (fire, flood, earthquake)
- Power outages
- Hardware failures
- Data center failures
- Climate control failures

**4. Technical Threats**
- Software vulnerabilities and exploits
- Unpatched systems
- Malware and viruses
- DDoS attacks
- SQL injection and XSS
- Man-in-the-middle attacks
- Zero-day exploits
- Insecure APIs

**5. Vendor/Third-Party Threats**
- Vendor data breaches
- Supply chain attacks
- Lack of vendor security
- Subcontractor vulnerabilities
- Third-party service outages

#### Threat Identification for Our Stack (NestJS + AWS + PostgreSQL)

**Backend (NestJS) Specific:**
- Dependency vulnerabilities (npm packages)
- Authentication bypass
- Authorization flaws (RBAC misconfiguration)
- API endpoint exposure
- Insecure session management
- Code injection vulnerabilities
- Insufficient input validation

**Database (PostgreSQL/RDS) Specific:**
- SQL injection attacks
- Insufficient access controls
- Unencrypted data at rest
- Unencrypted connections
- Missing audit logs
- Backup exposure
- Database credential theft

**Frontend (Next.js/React) Specific:**
- XSS vulnerabilities
- CSRF attacks
- Insecure data storage (localStorage)
- Exposed API keys
- Insecure HTTP connections
- Client-side logic bypass
- Session hijacking

**AWS Infrastructure Specific:**
- Misconfigured S3 buckets (public access)
- Overly permissive IAM policies
- Missing MFA on root accounts
- Unencrypted EBS volumes
- Security group misconfigurations
- Missing CloudTrail logs
- VPC misconfigurations

---

### Phase 4: Vulnerability Assessment

Identify weaknesses that could be exploited by threats.

#### Vulnerability Categories

**1. Technical Vulnerabilities**
- Unpatched software
- Outdated operating systems
- Known CVEs in dependencies
- Missing encryption
- Weak encryption algorithms (< TLS 1.2)
- Open ports and services
- Default credentials
- Insecure configurations

**2. Administrative Vulnerabilities**
- Lack of security policies
- Insufficient employee training
- No incident response plan
- Inadequate access controls
- Missing background checks
- Poor password policies
- No security awareness program

**3. Physical Vulnerabilities**
- Inadequate facility security
- No visitor logs
- Uncontrolled device access
- No workstation locks
- Improper equipment disposal
- No secure storage for backups

#### Vulnerability Assessment Methods

**Automated Scanning (Every 6 Months - 2025 Requirement):**

Tools to use:
- **AWS Inspector** - Scans EC2 instances for vulnerabilities
- **Nessus/OpenVAS** - Network vulnerability scanners
- **OWASP ZAP** - Web application security scanner
- **npm audit** - JavaScript dependency vulnerabilities
- **Snyk** - Continuous security scanning

```bash
# Example: Check npm dependencies for vulnerabilities
npm audit

# Example: Check for outdated packages
npm outdated
```

**Manual Review:**
- Configuration reviews (AWS, database, application)
- Code reviews (security-focused)
- Architecture analysis
- Access control reviews
- Policy and procedure reviews

**Penetration Testing (Annual - 2025 Requirement):**
- Hire external security firm
- Simulate real-world attacks
- Test all layers (network, application, physical)
- Document findings and remediation

---

### Phase 5: Risk Determination

Assess likelihood and impact for each threat/vulnerability combination.

#### Risk Matrix

Use this matrix to calculate risk levels:

| Likelihood → <br> ↓ Impact | **Rare** (1) | **Unlikely** (2) | **Possible** (3) | **Likely** (4) | **Almost Certain** (5) |
|------------|--------------|------------------|------------------|----------------|------------------------|
| **Catastrophic** (5) | Medium (5) | High (10) | High (15) | Critical (20) | Critical (25) |
| **Major** (4) | Low (4) | Medium (8) | High (12) | High (16) | Critical (20) |
| **Moderate** (3) | Low (3) | Medium (6) | Medium (9) | High (12) | High (15) |
| **Minor** (2) | Low (2) | Low (4) | Medium (6) | Medium (8) | High (10) |
| **Negligible** (1) | Low (1) | Low (2) | Low (3) | Low (4) | Medium (5) |

**Risk Score = Likelihood × Impact**

**Risk Levels:**
- **Critical (16-25):** Immediate action required
- **High (11-15):** Prioritize remediation
- **Medium (6-10):** Plan remediation within timeframe
- **Low (1-5):** Accept or monitor

#### Likelihood Scale

| Level | Score | Description | Frequency |
|-------|-------|-------------|-----------|
| **Almost Certain** | 5 | Expected to occur | > 90% probability |
| **Likely** | 4 | Will probably occur | 60-90% probability |
| **Possible** | 3 | Might occur | 30-60% probability |
| **Unlikely** | 2 | Not expected | 10-30% probability |
| **Rare** | 1 | May occur in exceptional circumstances | < 10% probability |

#### Impact Scale

| Level | Score | Description | Consequences |
|-------|-------|-------------|--------------|
| **Catastrophic** | 5 | Extreme impact | - Massive PHI breach (10,000+ records)<br>- Regulatory penalties > $1M<br>- Organization shutdown risk<br>- National media coverage |
| **Major** | 4 | Significant impact | - Large PHI breach (1,000-10,000 records)<br>- Penalties $250K-$1M<br>- Extended service outage (days)<br>- Significant reputation damage |
| **Moderate** | 3 | Noticeable impact | - Medium PHI breach (100-1,000 records)<br>- Penalties $50K-$250K<br>- Service outage (hours)<br>- Local media coverage |
| **Minor** | 2 | Limited impact | - Small PHI breach (10-100 records)<br>- Penalties $10K-$50K<br>- Brief service degradation<br>- Few customers affected |
| **Negligible** | 1 | Minimal impact | - Isolated incident (< 10 records)<br>- Minor penalties or warnings<br>- No service impact<br>- Contained internally |

#### Risk Assessment Template

| Risk ID | Threat | Vulnerability | Affected Asset | Likelihood (1-5) | Impact (1-5) | Risk Score | Risk Level | Current Controls | Residual Risk | Action Required |
|---------|--------|---------------|----------------|------------------|--------------|------------|------------|------------------|---------------|-----------------|
| R-001 | SQL Injection | Unvalidated inputs | API endpoints | 3 | 5 | 15 | High | Parameterized queries | 6 (Medium) | Add input validation layer |
| R-002 | Ransomware | Unpatched servers | EC2 instances | 2 | 4 | 8 | Medium | AWS patching | 4 (Low) | Implement automated patching |
| R-003 | Insider threat | Excessive permissions | Production DB | 3 | 4 | 12 | High | RBAC implemented | 6 (Medium) | Review and tighten permissions |
| R-004 | Device theft | No device encryption | Employee laptops | 2 | 3 | 6 | Medium | Password protection | 6 (Medium) | Enable full disk encryption |

**For Each Risk, Document:**
- Unique risk identifier
- Threat description
- Vulnerability exploited
- Asset(s) affected
- Likelihood rating (1-5) with justification
- Impact rating (1-5) with justification
- Risk score (likelihood × impact)
- Risk level (Critical/High/Medium/Low)
- Current security controls in place
- Residual risk after controls
- Recommended actions
- Priority for remediation
- Responsible party
- Target completion date

---

### Phase 6: Remediation Planning

Create action plan to address identified risks.

#### Remediation Strategies

**1. Avoid the Risk**
- Eliminate the activity causing the risk
- Example: Stop using non-HIPAA-compliant service

**2. Mitigate the Risk**
- Reduce likelihood or impact
- Example: Implement MFA, encryption, firewalls

**3. Transfer the Risk**
- Share risk with third party
- Example: Cybersecurity insurance, BAA with vendor

**4. Accept the Risk**
- Document decision to accept (must be low risk)
- Example: Accept risk of physical break-in at well-secured office

#### Prioritization Framework

**Critical Risks (Score 16-25):**
- **Timeline:** Immediate action (within 30 days)
- **Resources:** Allocate immediately
- **Approval:** Executive level
- **Example Actions:**
  - Patch critical vulnerabilities
  - Fix publicly exposed PHI
  - Implement missing encryption

**High Risks (Score 11-15):**
- **Timeline:** 30-90 days
- **Resources:** High priority
- **Approval:** Security Officer
- **Example Actions:**
  - Implement MFA
  - Fix authorization flaws
  - Update security policies

**Medium Risks (Score 6-10):**
- **Timeline:** 90-180 days
- **Resources:** Planned effort
- **Approval:** Team lead
- **Example Actions:**
  - Improve logging
  - Enhance monitoring
  - Staff training

**Low Risks (Score 1-5):**
- **Timeline:** 180+ days or accept
- **Resources:** As available
- **Approval:** Team lead
- **Example Actions:**
  - Minor process improvements
  - Nice-to-have enhancements
  - Documentation updates

#### Remediation Plan Template

| Risk ID | Risk Description | Priority | Remediation Action | Owner | Target Date | Status | Cost Estimate | Notes |
|---------|------------------|----------|-------------------|-------|-------------|--------|---------------|-------|
| R-001 | SQL injection risk | Critical | Implement input validation library | Backend Team | 2025-02-15 | In Progress | 40 hours | Using Zod for validation |
| R-003 | Excessive DB permissions | High | Review and restrict IAM roles | DevOps | 2025-03-01 | Planned | 16 hours | Coordinate with teams |
| R-007 | Missing MFA | High | Implement MFA for all users | IT | 2025-03-15 | Not Started | $200/mo | AWS Cognito MFA |
| R-012 | No device encryption | Medium | Deploy encryption policy | IT | 2025-04-30 | Planned | 8 hours | Use BitLocker/FileVault |

---

### Phase 7: Documentation

**Required Documentation:**

#### 1. Executive Summary (1-2 pages)
- Assessment scope and period
- Assessment team members
- Overall risk posture
- Key findings summary
- Critical/high risks count
- Top recommendations
- Resources required

#### 2. Detailed Risk Assessment Report
- Complete methodology
- Asset inventory
- Threat catalog
- Vulnerability findings
- Risk matrix with all risks
- Current security controls
- Residual risk after controls
- Detailed recommendations

#### 3. Remediation Plan
- Prioritized action items
- Responsible parties
- Timelines
- Resource requirements
- Budget estimates
- Success criteria

#### 4. Supporting Documentation
- Scan results (vulnerability scans)
- Penetration test reports
- Configuration reviews
- Policy reviews
- Interview notes
- Network diagrams
- Data flow diagrams

#### 5. Sign-off and Approval
- Security Officer signature
- Privacy Officer signature
- Executive management approval
- Date of completion

**Retention:** Keep all risk assessment documentation for **minimum 6 years** per HIPAA requirements.

---

## Risk Assessment Template (Downloadable Format)

### Basic Risk Assessment Worksheet

```markdown
# HIPAA Security Risk Assessment
**Organization:** [Your Company Name]
**Assessment Date:** [YYYY-MM-DD]
**Assessment Team:** [Names and Roles]
**Scope:** [Systems/Locations Covered]

---

## 1. ASSET INVENTORY

### Systems Containing ePHI

| ID | Asset Name | Type | Location | ePHI? | Sensitivity | Owner |
|----|------------|------|----------|-------|-------------|-------|
| A-001 | | | | | | |
| A-002 | | | | | | |
| A-003 | | | | | | |

---

## 2. THREAT IDENTIFICATION

### Identified Threats

| ID | Threat Category | Threat Description | Affected Assets |
|----|----------------|-------------------|-----------------|
| T-001 | | | |
| T-002 | | | |
| T-003 | | | |

---

## 3. VULNERABILITY ASSESSMENT

### Identified Vulnerabilities

| ID | Vulnerability | Systems Affected | How Discovered |
|----|--------------|------------------|----------------|
| V-001 | | | |
| V-002 | | | |
| V-003 | | | |

---

## 4. RISK ANALYSIS

### Risk Register

| Risk ID | Threat | Vulnerability | Asset | L | I | Score | Level | Current Controls | Residual Risk |
|---------|--------|---------------|-------|---|---|-------|-------|------------------|---------------|
| R-001 | | | | | | | | | |
| R-002 | | | | | | | | | |
| R-003 | | | | | | | | | |

**Legend:**
- L = Likelihood (1-5)
- I = Impact (1-5)
- Score = L × I
- Level = Critical/High/Medium/Low

---

## 5. REMEDIATION PLAN

### Action Items

| Risk ID | Action | Priority | Owner | Target Date | Status |
|---------|--------|----------|-------|-------------|--------|
| R-001 | | | | | |
| R-002 | | | | | |
| R-003 | | | | | |

---

## 6. APPROVALS

**Security Officer:**
- Name: _______________________________
- Signature: _______________________________
- Date: _______________________________

**Privacy Officer:**
- Name: _______________________________
- Signature: _______________________________
- Date: _______________________________

**Executive Sponsor:**
- Name: _______________________________
- Signature: _______________________________
- Date: _______________________________

---

**Next Assessment Due:** [YYYY-MM-DD] (Annual)
```

---

## Common Mistakes in Risk Assessments

### ❌ Mistake 1: Incomplete Asset Inventory

**Problem:** Missing systems, especially third-party services

**Impact:** Unassessed risks, compliance gaps

**Solution:**
- ✅ Catalog ALL systems that touch ePHI
- ✅ Include third-party services and vendors
- ✅ Document shadow IT discoveries
- ✅ Maintain continuous inventory

---

### ❌ Mistake 2: Generic Threats Only

**Problem:** Using generic threat lists without customization

**Impact:** Miss specific risks to your technology stack

**Solution:**
- ✅ Tailor threats to your specific environment
- ✅ Consider your tech stack (NestJS, AWS, PostgreSQL)
- ✅ Include threats relevant to your industry
- ✅ Review recent breach reports for your sector

---

### ❌ Mistake 3: No Actual Testing

**Problem:** Paper-only assessment without real vulnerability testing

**Impact:** Unknown vulnerabilities, false sense of security

**Solution:**
- ✅ Run automated vulnerability scans
- ✅ Conduct penetration testing (required annually)
- ✅ Review configurations manually
- ✅ Test security controls

---

### ❌ Mistake 4: Ignoring Residual Risk

**Problem:** Not assessing risk AFTER controls are in place

**Impact:** Over-confidence in security posture

**Solution:**
- ✅ Calculate residual risk after controls
- ✅ Ensure residual risk is acceptable
- ✅ Document acceptance of remaining risk
- ✅ Plan additional controls if needed

---

### ❌ Mistake 5: Inadequate Documentation

**Problem:** Incomplete or missing documentation

**Impact:** Cannot prove compliance in audit

**Solution:**
- ✅ Document everything thoroughly
- ✅ Keep supporting evidence (scan reports, logs)
- ✅ Maintain version history
- ✅ Store securely for 6+ years

---

### ❌ Mistake 6: One-and-Done Approach

**Problem:** Treating assessment as checkbox exercise

**Impact:** Outdated risk profile, new vulnerabilities unaddressed

**Solution:**
- ✅ Schedule annual assessments (2025 requirement)
- ✅ Reassess after major changes
- ✅ Continuous monitoring
- ✅ Update remediation plan regularly

---

## Tools and Resources

### Official HIPAA Resources

**HHS Security Risk Assessment Tool:**
- **What:** Free downloadable tool from HealthIT.gov
- **Use:** Structured questionnaire and risk calculation
- **Format:** Excel-based
- **Link:** [HealthIT.gov SRA Tool](https://www.healthit.gov/topic/privacy-security-and-hipaa/security-risk-assessment-tool)
- **Note:** Use optional, but helpful for small organizations

**HHS Risk Analysis Guidance:**
- Official guidance on conducting risk analysis
- [Guidance on Risk Analysis](https://www.hhs.gov/hipaa/for-professionals/security/guidance/guidance-risk-analysis/index.html)

### Vulnerability Scanning Tools

**For AWS Infrastructure:**
- **AWS Inspector** - Automated security assessment for EC2
- **AWS Security Hub** - Centralized security findings
- **AWS Config** - Configuration compliance monitoring
- **AWS GuardDuty** - Threat detection

**For Applications:**
- **OWASP ZAP** - Web application scanner (free, open source)
- **Burp Suite** - Professional web security testing
- **npm audit** - Node.js dependency vulnerabilities
- **Snyk** - Continuous dependency scanning

**Network Scanning:**
- **Nessus** - Comprehensive vulnerability scanner
- **OpenVAS** - Open source vulnerability scanner
- **Qualys** - Cloud-based scanning platform

### Penetration Testing Services

**When to Hire External Firm:**
- Annual requirement (2025)
- Before major launches
- After significant changes
- Following security incidents

**What to Look For:**
- Healthcare industry experience
- HIPAA-specific testing
- Comprehensive reporting
- Remediation guidance
- Compliance-focused approach

---

## Tech Stack-Specific Risk Assessment

### NestJS Backend Risks

**Common Vulnerabilities:**
- [ ] Outdated npm dependencies with known CVEs
- [ ] Missing input validation on API endpoints
- [ ] Insufficient authentication/authorization checks
- [ ] Exposed sensitive information in error messages
- [ ] Missing rate limiting on endpoints
- [ ] CORS misconfiguration
- [ ] Insufficient session security
- [ ] Inadequate audit logging

**Assessment Actions:**
```bash
# Check for vulnerable dependencies
npm audit

# Check for outdated packages
npm outdated

# Use security linters
npm install -D eslint-plugin-security
```

---

### PostgreSQL/RDS Database Risks

**Common Vulnerabilities:**
- [ ] Unencrypted data at rest
- [ ] Unencrypted connections (no SSL)
- [ ] Weak or default credentials
- [ ] Overly permissive access controls
- [ ] Public accessibility
- [ ] Missing audit logging (pgAudit not enabled)
- [ ] No automated backups
- [ ] Unencrypted backups

**Assessment Actions:**
- Review RDS encryption settings
- Verify SSL/TLS enforcement
- Audit IAM policies and database users
- Check security group rules
- Enable Enhanced Monitoring
- Enable pgAudit extension

---

### Next.js/React Frontend Risks

**Common Vulnerabilities:**
- [ ] XSS vulnerabilities
- [ ] Insecure API key storage
- [ ] PHI stored in localStorage/sessionStorage
- [ ] Insecure HTTP connections
- [ ] Missing Content Security Policy
- [ ] CSRF vulnerabilities
- [ ] Exposed source maps in production
- [ ] Console.log() statements with PHI

**Assessment Actions:**
- Review data storage mechanisms
- Audit API communications
- Check for hardcoded secrets
- Review CSP headers
- Test for XSS/CSRF
- Verify HTTPS enforcement

---

### AWS Infrastructure Risks

**Common Vulnerabilities:**
- [ ] Public S3 buckets
- [ ] Overly permissive IAM policies
- [ ] Missing MFA on root account
- [ ] Unencrypted EBS volumes
- [ ] Missing CloudTrail logging
- [ ] Security groups allow 0.0.0.0/0
- [ ] No VPC flow logs
- [ ] Unused IAM credentials

**Assessment Actions:**
```bash
# Check for public S3 buckets
aws s3api list-buckets --query "Buckets[].Name" | xargs -I {} aws s3api get-bucket-acl --bucket {}

# List IAM users without MFA
aws iam get-credential-report

# Review security groups
aws ec2 describe-security-groups --query 'SecurityGroups[?IpPermissions[?IpRanges[?CidrIp==`0.0.0.0/0`]]]'
```

Use AWS Config Rules and Security Hub for automated monitoring.

---

## Sample Risk Scenarios for Healthcare Applications

### Scenario 1: Data Breach via SQL Injection

**Risk:** SQL injection vulnerability in patient search API

**Threat:** External attacker exploits unvalidated input

**Vulnerability:** Missing input sanitization in search endpoint

**Asset:** Production database with 50,000 patient records

**Likelihood:** 3 (Possible - vulnerability exists, but requires specific knowledge)

**Impact:** 5 (Catastrophic - massive PHI breach, regulatory penalties)

**Risk Score:** 15 (High)

**Current Controls:** Parameterized queries in most endpoints

**Remediation:**
1. Implement input validation library (Zod) - 2 weeks
2. Security audit all API endpoints - 1 week
3. Deploy WAF rules for SQL injection patterns - 1 week
4. Add automated security testing to CI/CD - 2 weeks

**Residual Risk:** 6 (Medium) after all controls

---

### Scenario 2: Insider Access Abuse

**Risk:** Employee accessing patient records inappropriately

**Threat:** Malicious or curious insider

**Vulnerability:** Lack of audit trail review, excessive permissions

**Asset:** Production database

**Likelihood:** 2 (Unlikely - employees trained, but possible)

**Impact:** 3 (Moderate - could affect dozens of patients before detection)

**Risk Score:** 6 (Medium)

**Current Controls:** RBAC implemented, audit logging enabled

**Remediation:**
1. Implement automated anomaly detection for unusual access - 4 weeks
2. Weekly audit log reviews - ongoing
3. Reduce permissions to minimum necessary - 1 week
4. Implement "break glass" emergency access logging - 2 weeks

**Residual Risk:** 2 (Low) after all controls

---

## What's Next?

After completing your risk assessment:

1. **[Third-Party Vendors](04-third-party-vendors.md)** - Evaluate vendor security as part of your risk profile
2. **[Backend Playbook](../03-backend-playbook/README.md)** - Implement technical safeguards to address identified risks
3. **[DevOps Playbook](../05-devops-playbook/README.md)** - Secure your infrastructure based on risk findings

---

## Additional Resources

**Official HIPAA Resources:**
- [HHS Risk Analysis Guidance](https://www.hhs.gov/hipaa/for-professionals/security/guidance/guidance-risk-analysis/index.html) - Official methodology
- [HealthIT.gov SRA Tool](https://www.healthit.gov/topic/privacy-security-and-hipaa/security-risk-assessment-tool) - Free risk assessment tool
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework) - Risk management framework

**Risk Assessment Guides:**
- [HIPAA Journal - Risk Assessment Guide](https://www.hipaajournal.com/hipaa-risk-assessment/) - Comprehensive overview
- [OCR Security Rule Guidance](https://www.hhs.gov/hipaa/for-professionals/security/guidance/index.html) - Official security guidance

**Vulnerability Management:**
- [HIPAA Vulnerability Scan Requirements](https://beaglesecurity.com/blog/article/hipaa-vulnerability-scan-requirements.html) - 2025 scanning requirements
- [OWASP Top 10](https://owasp.org/www-project-top-ten/) - Common web application risks

---

*Last Updated: November 2025*
*Next: [Third-Party Vendors →](04-third-party-vendors.md)*
