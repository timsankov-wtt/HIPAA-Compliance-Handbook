---
layout: default
title: 2025 HIPAA Security Rule Updates
parent: HIPAA Fundamentals
nav_order: 3
---

# 2025 HIPAA Security Rule Updates

## Overview

In January 2025, HHS published a Notice of Proposed Rulemaking (NPRM) for the HIPAA Security Rule - the first major update in nearly 20 years. The final rule is expected late 2025 or early 2026. These proposed changes transform cybersecurity best practices from recommended to mandatory requirements.

**Status**: Proposed rule (comment period closed March 2025). Organizations should prepare for implementation within 12 months of final rule publication.

## Proposed Changes

### 1. Mandatory Encryption

- **Previous**: "Addressable" specification (optional if alternative controls used)
- **Proposed**: Required for all ePHI at rest and in transit
- **Approved Algorithms**:
  - Data at rest: AES-256
  - Data in transit: TLS 1.3
  - Key exchanges: RSA-2048 (minimum)

### 2. Multi-Factor Authentication (MFA)

- **Scope**: All access to ePHI (not just remote access)
- **Requirements**: At least two of three factors:
  - Knowledge (password, PIN)
  - Possession (token, smart card)
  - Inherence (fingerprint, facial recognition)
- **Limited exceptions** allowed (must be documented)

### 3. Removal of "Addressable" vs "Required" Distinction

- All implementation specifications become required with limited, documented exceptions
- Clarifies that specifications were never truly optional
- Risk assessment required for any exceptions

### 4. Asset Inventory

- Continuous, updated inventory of all assets that create, receive, maintain, or transmit ePHI
- Remove unauthorized or extraneous software

### 5. Regular Security Testing

- **Vulnerability Scanning**: Every 6 months (minimum)
- **Penetration Testing**: Annually (minimum)
- Documentation of findings and remediation required

## Technical Implementation

### Backend (NestJS)

- Encryption middleware for all API responses with ePHI
- MFA support in authentication flows
- Comprehensive audit logging for ePHI access
- Session management with MFA validation

### Database (PostgreSQL)

- Transparent Data Encryption (TDE) or column-level encryption
- Row-level security for ePHI tables
- Encrypted backups with key rotation
- Database activity monitoring and logging

### Infrastructure (AWS)

- AWS KMS for encryption key management
- TLS 1.3 for all data in transit
- VPC segmentation and security groups
- AWS Config and GuardDuty for compliance monitoring
- Regular vulnerability scans and penetration tests

## Resources

- [HHS HIPAA Security Rule NPRM](https://www.hhs.gov/hipaa/for-professionals/security/hipaa-security-rule-nprm/factsheet/index.html)
- [NIST SP 800-66r2](https://csrc.nist.gov/publications/detail/sp/800-66/rev-2/final)
- [AWS HIPAA Compliance](https://aws.amazon.com/compliance/hipaa-compliance/)
