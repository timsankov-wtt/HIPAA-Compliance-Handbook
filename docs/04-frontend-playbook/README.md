# Frontend Playbook

> Building HIPAA-aware user interfaces with Next.js/React and Flutter

---

## Frontend Responsibilities

The frontend is the **first and last place** where users interact with Protected Health Information (PHI). Even with a secure backend, a weak UI can leak data through:

- Screenshots, screen sharing, and shoulder-surfing
- Browser storage and dev tools
- Misconfigured CORS or insecure HTTP calls
- Overly verbose error messages and logs

As a frontend engineer, you are responsible for:

- Ensuring PHI is **never exposed unnecessarily** in the browser or app
- Enforcing **secure client behaviour** (validation, sanitization, transport)
- Designing **privacy-aware UX** that respects patients’ expectations
- Integrating cleanly with backend authentication, authorization, and logging

---

## Why Frontend Security Matters

HIPAA is often seen as a “backend problem”, but many real breaches start on the client:

- PHI stored in `localStorage` or logs and later exfiltrated
- Sensitive data displayed to the wrong role because of missing UI checks
- PHI leaked via query parameters, error messages, or analytics tools

A compliant system requires the **frontend and backend** to work together:

- Backend enforces **authoritative access control** and audit logging
- Frontend enforces **good hygiene** and prevents accidental exposure

---

## Technology Scope

This playbook focuses on:

- **Web**: Next.js/React (SPA, SSR, and App Router patterns)
- **Mobile**: Flutter (iOS, Android)

The principles apply to other frameworks (Vue, Angular, React Native), but examples are shown using these technologies.

---

## Key Principles for Frontend

- **Never trust client-side checks**  
  All security decisions must be validated server-side. Frontend validation is for UX and basic safety only.

- **Secure data transmission**  
  Always send PHI over **HTTPS/TLS**, use secure cookie storage where possible, and avoid exposing PHI in URLs or logs.

- **Privacy-aware UI/UX**  
  Show only the **minimum necessary** PHI, use masking where appropriate, and avoid displaying sensitive data by default.

- **Robust session management**  
  Respect backend session timeouts, handle token expiry gracefully, and log users out after inactivity.

---

## Navigation

Use this playbook together with the Backend and DevOps playbooks.

- **Data Validation & Sanitization** → `01-data-validation.md`
- **Secure Transmission** → `02-secure-transmission.md`
- **UI Security & Privacy UX** → `03-ui-security.md`
- **Frontend Checklist (Pre-deploy)** → `04-checklist.md`

For backend requirements, see: `../03-backend-playbook/README.md` and related sections.

---

## Quick 10‑Item Frontend Checklist

Use this as a fast pre-commit or pre-deploy reminder. Full details are in `04-checklist.md`.

- [ ] **No PHI in localStorage/sessionStorage or URL/query parameters**
- [ ] **All API calls to PHI endpoints use HTTPS and authenticated sessions**
- [ ] **Client-side validation does not replace server-side validation**
- [ ] **Dangerous HTML rendering (`dangerouslySetInnerHTML`) is audited and sanitized**
- [ ] **No PHI appears in console logs or analytics tools**
- [ ] **UI only shows PHI appropriate to the current user’s role**
- [ ] **Sensitive fields (SSN, DOB, identifiers) are masked where appropriate**
- [ ] **Session timeout and re-auth flows are tested end-to-end**
- [ ] **CORS configuration only allows trusted origins for authenticated requests**
- [ ] **Flutter apps pin certificates / validate TLS and use secure storage APIs**
