---
layout: default
title: Understanding Protected Health Information (PHI)
parent: HIPAA Fundamentals
nav_order: 2
---

# Understanding Protected Health Information (PHI)

> A comprehensive guide to identifying and handling PHI in your applications

---

## What is PHI?

**Protected Health Information (PHI)** is any information in a medical record or other health-related information that can be used to identify an individual and that was created, used, or disclosed in the course of providing healthcare services.

### The 18 PHI Identifiers

PHI includes any health information that is paired with any of these 18 identifiers:

1. **Names**

   - Full names (first, last, or initial + last)
   - Maiden names
   - Aliases

2. **Geographical identifiers**

   - Geographic subdivisions smaller than a state
   - Street address, city, county, precinct, ZIP code
   - First 3 digits of ZIP if geographic unit contains < 20,000 people

3. **Dates**

   - Birth date
   - Admission date
   - Discharge date
   - Date of death
   - Age over 89 (must be aggregated into 5-year ranges)

4. **Phone numbers**

   - Home, work, mobile
   - Fax numbers

5. **Email addresses**

   - Personal email
   - Work email (if it contains the person's name)

6. **Social Security numbers**

   - Full or last 4 digits in some contexts

7. **Medical record numbers**

   - Patient IDs
   - Case numbers

8. **Health plan beneficiary numbers**

   - Insurance member IDs
   - Policy numbers

9. **Account numbers**

   - Billing accounts
   - Credit card numbers (when used for payment)

10. **Certificate/license numbers**

    - Driver's license
    - Medical license

11. **Vehicle identifiers**

    - License plate numbers
    - Vehicle serial numbers

12. **Device identifiers and serial numbers**

    - Medical device IDs
    - Implant serial numbers

13. **Web URLs**

    - Personal website URLs
    - IP addresses

14. **IP addresses**

    - IPv4 and IPv6 addresses

15. **Biometric identifiers**

    - Fingerprints
    - Retina scans
    - Voiceprints

16. **Full-face photos and comparable images**

    - Patient photos
    - X-rays
    - MRIs

17. **Any other unique identifying number, characteristic, or code**
    - Unique codes in research studies
    - Pseudonymized data if the key exists

## ePHI: Electronic PHI

ePHI is any PHI that is created, stored, transmitted, or received electronically. This includes:

- **Databases** containing patient information
- **Emails** with health information
- **Cloud storage** with medical records
- **Mobile apps** that collect health data
- **Wearable devices** that track health metrics

### Examples of ePHI in Applications

```typescript
// Example 1: Database record
{
  id: "12345",
  firstName: "John",
  lastName: "Doe",
  dob: "1985-04-23",
  ssn: "***-**-1234",
  diagnosis: "Type 2 Diabetes",
  lastAppointment: "2025-03-15"
}

// Example 2: API Response
{
  "patientId": "P123456",
  "email": "johndoe@example.com",
  "phone": "(555) 123-4567",
  "conditions": ["Hypertension", "Hyperlipidemia"],
  "medications": ["Lisinopril 10mg daily"]
}
```

## PHI vs Non-PHI

### PHI Examples

- John Smith's blood pressure reading from his doctor's visit
- An email containing a patient's lab results
- A database record linking a patient's name with their diagnosis
- A prescription label with patient name and medication

### Non-PHI Examples

- Aggregated health statistics (e.g., "30% of patients have high blood pressure")
- De-identified health data (with all 18 identifiers removed)
- Step count from a fitness tracker (unless linked to identifiable information)
- Employment records that include health insurance information (covered by other laws, not HIPAA)

## Common PHI in Applications

### Web Forms

```html
<!-- PHI in form fields -->
<form id="patientIntake">
  <input type="text" name="fullName" placeholder="Full Name" />
  <input type="date" name="birthDate" />
  <input type="tel" name="phone" />
  <textarea name="medicalHistory"></textarea>
</form>
```

### API Endpoints

```typescript
// PHI in API routes
app.get("/api/patients/:id/records", authenticateUser, async (req, res) => {
  // Verify user has access to this patient's records
  const hasAccess = await checkAccess(req.user.id, req.params.id);
  if (!hasAccess) {
    return res.status(403).json({ error: "Access denied" });
  }

  // Return PHI
  const records = await getMedicalRecords(req.params.id);
  res.json(records);
});
```

## Best Practices for Handling PHI

1. **Minimize Collection**

   - Only collect PHI that's absolutely necessary
   - Consider if you can use de-identified data instead

2. **Secure Storage**

   - Encrypt PHI at rest and in transit
   - Use strong access controls
   - Implement proper backup procedures

3. **Careful Logging**

   - Avoid logging full PHI in application logs
   - Implement proper log redaction

4. **Secure Transmission**

   - Use TLS 1.2+ for all data in transit
   - Never send PHI via unencrypted email

5. **Proper Disposal**
   - Securely delete PHI when no longer needed
   - Follow data retention policies

## Common Pitfalls

1. **Logging PHI**

   ```javascript
   // ❌ Bad: Logging PHI
   console.log(`Processing record for ${patient.name}, SSN: ${patient.ssn}`);

   // ✅ Better: Log minimal, non-identifying information
   console.log(`Processing record for patient ID: ${patient.id}`);
   ```

2. **URL Parameters**

   ```
   // ❌ Bad: PHI in URL
   /patient-records?name=John+Doe&dob=1985-04-23

   // ✅ Better: Use IDs instead
   /api/patients/12345/records
   ```

3. **Error Messages**

   ```javascript
   // ❌ Bad: Revealing PHI in errors
   throw new Error(`Patient ${name} with SSN ${ssn} not found`);

   // ✅ Better: Generic error messages
   throw new Error("Patient record not found");
   ```

## Testing for PHI Leaks

1. **Code Reviews**

   - Check for hardcoded PHI in test data
   - Verify logging practices
   - Review API responses

2. **Automated Scans**

   - Use tools to scan for SSNs, credit card numbers, etc.
   - Check for PHI in logs and error messages

3. **Penetration Testing**
   - Test for PHI exposure in:
     - API responses
     - Error messages
     - Log files
     - Client-side storage

## Resources

- [HHS Guidance on De-identification of PHI](https://www.hhs.gov/hipaa/for-professionals/privacy/special-topics/de-identification/index.html)
- [NIST Guide to Protecting the Confidentiality of PII](https://csrc.nist.gov/publications/detail/sp/800-122/final)
- [OWASP Top 10 Privacy Risks](https://owasp.org/www-pdf-archive/OWASP_Top_10_Privacy_Risks.pdf)

---

**Next Steps:** [HIPAA 2025 Updates](../01-fundamentals/03-2025-updates.md)
