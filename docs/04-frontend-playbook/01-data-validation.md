---
layout: default
title: Data Validation & Sanitization
parent: Frontend Playbook
nav_order: 2
---

# Data Validation & Sanitization

> Frontend patterns to prevent injection and data exposure before it reaches the backend

---

## 1. Why Frontend Validation Matters (But Is Not Enough)

Frontend validation provides:

- **Better UX** – instant feedback instead of round-trips for obvious errors
- **Basic protection** – reduces accidental injection or malformed data
- **Defense in depth** – makes exploitation harder even if backend has gaps

However, for HIPAA:

- **Server-side validation is always mandatory.**
- Frontend validation is a **first filter**, not the final decision-maker.
- Assume an attacker can bypass the UI and send raw HTTP requests.

Design rule:

> Treat frontend validation as **UX and hygiene**, backend validation as **security and compliance**.

---

## 2. Input Validation

### 2.1 Client-Side Validation Patterns

In both React and Flutter:

- Validate **shape** (required fields, type, format)
- Validate **range** (length, allowed values, min/max)
- Avoid free‑text when a constrained option (dropdown, radio) is possible

**React + React Hook Form + Zod (example)**

```typescript
import { useForm } from "react-hook-form";
import { z } from "zod";
import { zodResolver } from "@hookform/resolvers/zod";

const patientSchema = z.object({
  fullName: z.string().min(1, "Name is required"),
  birthDate: z.string().regex(/\d{4}-\d{2}-\d{2}/, "Use YYYY-MM-DD"),
  phone: z.string().min(10).max(20),
  reasonForVisit: z.string().max(500), // Prevent overly long input
});

type PatientForm = z.infer<typeof patientSchema>;

export function IntakeForm() {
  const { register, handleSubmit, formState } = useForm<PatientForm>({
    resolver: zodResolver(patientSchema),
  });

  const onSubmit = (data: PatientForm) => {
    // Send to backend after validation
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {/* inputs with {...register("fullName")} etc. */}
    </form>
  );
}
```

### 2.2 Why Server-Side Validation Is Mandatory

Even with strong client validation, attackers can:

- Use browser dev tools to change JavaScript
- Replay or modify requests via tools like Postman
- Bypass your SPA completely and hit the API

Therefore, the backend must **re-validate** everything and reject invalid or unexpected data, especially where PHI is involved.

---

## 3. XSS Prevention

### 3.1 React/Next.js Defaults

React helps by **escaping values in JSX** by default:

```tsx
// Safe: React escapes `patientName`
<h1>Hello, {patientName}</h1>
```

XSS problems usually appear when you **opt out** of this behavior.

### 3.2 Dangerous HTML Rendering

```tsx
// Potentially unsafe if `htmlFromBackend` is not sanitized
<div dangerouslySetInnerHTML={{ __html: htmlFromBackend }} />
```

If you must render HTML:

- Sanitize on the **backend** and
- Optionally sanitize again on the **frontend** with a library like **DOMPurify**.

```tsx
import DOMPurify from "dompurify";

const safeHtml = DOMPurify.sanitize(htmlFromBackend);
<div dangerouslySetInnerHTML={{ __html: safeHtml }} />;
```

### 3.3 Content Security Policy (CSP)

Configure CSP headers (usually via backend / CDN) to:

- Disallow inline scripts and `eval`
- Limit script and style sources to trusted origins
- Reduce impact of potential XSS vulnerabilities

---

## 4. Data Sanitization

Sanitization ensures that data is in a **safe and expected format** before sending it to the backend or displaying it.

Typical sanitizations:

- Trimming whitespace
- Normalizing phone numbers and dates
- Stripping HTML tags from user-generated text (where appropriate)

**Example using `validator.js` before submit:**

```typescript
import validator from "validator";

function sanitizeIntakePayload(raw: any) {
  return {
    fullName: validator.trim(raw.fullName || ""),
    email: validator.normalizeEmail(raw.email || "") || undefined,
    phone: validator.trim(raw.phone || ""),
    notes: validator.stripLow(raw.notes || "", true),
  };
}
```

### 4.1 Before Sending to Backend

- Normalize and constrain values (e.g., date formats)
- Remove unexpected extra fields from payloads
- Avoid sending unused PHI fields (minimum necessary principle)

### 4.2 Before Displaying

- Escape or sanitize strings coming from the backend
- Avoid rendering raw JSON or debug output containing PHI

---

## 5. React/Next.js Patterns

### 5.1 Form Validation Flow

- Use **schema-based validation** (Zod, Yup) shared between frontend and backend when possible.
- Display user-friendly errors without exposing internal details.

### 5.2 API Client Validation

Even if the form is valid, the **API response** might not be. For security and robustness:

- Validate API responses from the backend (e.g., Zod parsing of `fetch` responses)
- Fail closed: if data is malformed, show a generic error instead of rendering broken or partial PHI

```typescript
const patientResponseSchema = z.object({
  id: z.string(),
  displayName: z.string(),
  nextAppointment: z.string().nullable(),
});

async function fetchPatient(id: string) {
  const res = await fetch(`/api/patients/${id}`);
  const json = await res.json();
  return patientResponseSchema.parse(json); // throws if malformed
}
```

---

## 6. Flutter Considerations

### 6.1 Form Validation

Use `Form` widgets with validators instead of ad-hoc checks:

```dart
final _formKey = GlobalKey<FormState>();

Form(
  key: _formKey,
  child: Column(
    children: [
      TextFormField(
        decoration: const InputDecoration(labelText: 'Full name'),
        validator: (value) {
          if (value == null || value.trim().isEmpty) {
            return 'Name is required';
          }
          return null;
        },
      ),
      // ...
    ],
  ),
);
```

### 6.2 Input Sanitization

- Trim values before sending to the backend
- Restrict input types (e.g., `TextInputType.number` for numeric fields)
- Use input formatters to enforce patterns (phone numbers, dates)

---

## 7. Common Mistakes to Avoid

- **Relying only on frontend validation** – backend must re-validate all inputs.
- **Allowing rich HTML input without sanitization** – leads to XSS risks.
- **Over-collecting PHI in forms** – violate minimum necessary principle.
- **Logging raw form values containing PHI to console or analytics.**
- **Not validating API responses** – may render unexpected or malicious data.

---

## Quick Reference Checklist

- [ ] All forms validate required fields and basic formats on the client
- [ ] Dangerous HTML rendering is avoided or strictly sanitized
- [ ] Data is trimmed/normalized before sending to the backend
- [ ] Frontend collects only PHI needed for the current workflow
- [ ] No PHI is logged in console, error overlays, or analytics
- [ ] Backend re-validates everything; frontend does not rely on its own checks
