# Frontend Compliance Checklist

> Use this checklist before releasing any frontend that can display or submit PHI.

---

## How to Use This Checklist

- Run this checklist for **every major release** of your web or mobile frontend.
- Mark each item as:
  - **Checked** – implemented and verified
  - **N/A** – not applicable (document why elsewhere)
- For unchecked items, create a **remediation plan** before go‑live.

This checklist is organized into:

- Data Validation & Sanitization
- Secure Transmission
- UI Security
- React/Next.js Specific
- Flutter Specific
- Development & Build

---

## Critical

### Data Validation & Sanitization

- [ ] **Client‑side validation on all PHI forms**  
       **Validate:** Required fields, formats (e.g., dates, phone, email), and length limits are enforced; invalid input shows clear, non‑technical errors.

- [ ] **Frontend collects only minimum necessary PHI**  
       **Validate:** Forms do not ask for SSN, full medical history, or other sensitive data unless needed for the current workflow.

- [ ] **No unsafe HTML rendering of user‑generated content**  
       **Validate:** Any use of `dangerouslySetInnerHTML` or similar APIs is reviewed and sanitized with a library such as DOMPurify.

### Secure Transmission

- [ ] **All frontend API calls that involve PHI use HTTPS/TLS 1.2+**  
       **Validate:** Network tab in dev tools shows only `https://` requests; no mixed content warnings.

- [ ] **No PHI in URLs or query parameters**  
       **Validate:** Navigate through key flows and inspect URL bar; no names, DOBs, diagnoses, or other PHI values appear.

- [ ] **Tokens and sessions are handled securely**  
       **Validate:** Web apps use httpOnly secure cookies (or equivalent secure storage); Flutter apps do not store tokens in plain text.

### UI Security

- [ ] **Sensitive fields are masked by default**  
       **Validate:** SSN‑like identifiers, full DOB, and contact details are partially masked or hidden until explicitly revealed.

- [ ] **Role‑based UI prevents unauthorized visibility of PHI**  
       **Validate:** Log in as different roles and confirm that restricted screens, widgets, and actions are not visible.

- [ ] **User‑facing errors contain no PHI**  
       **Validate:** Force typical errors (missing record, access denied, server error) and verify no patient names or details appear.

### Development & Build

- [ ] **No PHI in console logs or client‑side debug output**  
       **Validate:** Inspect console during key workflows; confirm logs use IDs or correlation IDs instead of PHI.

- [ ] **Production build served with proper caching rules for PHI pages**  
       **Validate:** Sensitive pages are not cached in shared proxies/CNs in a way that would leak PHI between users.

---

## Important

### Data Validation & Sanitization

- [ ] **Schema‑based validation shared between frontend and backend where possible**  
       **Validate:** APIs validate the same constraints as the frontend (e.g., shared Zod/Yup schemas or documented contracts).

- [ ] **API responses are sanity‑checked before rendering**  
       **Validate:** Malformed or unexpected responses result in generic error screens, not partially broken PHI views.

### Secure Transmission

- [ ] **CORS configuration restricts origins to trusted frontends**  
       **Validate:** Authenticated endpoints do not use `*` for `Access-Control-Allow-Origin`; only approved domains are listed.

- [ ] **Session timeout and re‑authentication flows are implemented**  
       **Validate:** Idle sessions expire according to policy and require the user to re‑authenticate before viewing PHI again.

### UI Security

- [ ] **Show/Hide toggles for highly sensitive details**  
       **Validate:** Revealing full identifiers requires a deliberate user action (click/tap) and can be hidden again.

- [ ] **Permission‑aware navigation**  
       **Validate:** Menus and navigation links only include routes the current role may access.

### React/Next.js Specific

- [ ] **Next.js API routes touching PHI require authentication**  
       **Validate:** Routes under `/api` that return PHI reject anonymous requests and check user roles.

- [ ] **No PHI embedded into SSR HTML that might be cached**  
       **Validate:** Inspect initial HTML payloads; sensitive data is loaded via authenticated client‑side calls instead of being in static markup.

### Flutter Specific

- [ ] **Routing guards protect PHI screens**  
       **Validate:** Attempt to deep‑link into PHI screens without a valid session; app redirects to login or an error screen.

- [ ] **On‑device storage avoids PHI where possible**  
       **Validate:** No raw PHI is written to shared preferences, local files, or unencrypted databases.

### Development & Build

- [ ] **Source maps and debug info minimized in production**  
       **Validate:** Production bundles do not expose internal comments or debug strings that reference PHI or internal endpoints.

- [ ] **Third‑party tools reviewed for PHI handling**  
       **Validate:** Analytics, error tracking, and A/B testing tools are configured so no PHI is included in events.

---

## Recommended

### Data Validation & Sanitization

- [ ] **Client‑side input masks for common fields (phone, DOB)**  
       **Validate:** Inputs guide users into valid formats, reducing errors and cleanup work.

### Secure Transmission

- [ ] **Certificate pinning for mobile apps**  
       **Validate:** Simulated TLS interception or invalid certificates are rejected by the app.

### UI Security

- [ ] **Privacy‑aware design reviews**  
       **Validate:** UX/design reviews explicitly consider PHI exposure, masking, and minimum necessary disclosure.

### React/Next.js Specific

- [ ] **Centralized permission utilities (`useCan`, HOCs)**  
       **Validate:** All sensitive components check permissions using shared helpers, not ad‑hoc role checks.

### Flutter Specific

- [ ] **Centralized permission/role model in app state**  
       **Validate:** Widgets do not implement their own role logic; permissions come from a single, testable source of truth.

### Development & Build

- [ ] **Automated UI tests for PHI exposure scenarios**  
       **Validate:** E2E tests verify that unauthorized users cannot see PHI and that masking/timeout behavior works as intended.

---

## Sign‑Off

Use this section to document final approval before go‑live.

- **Frontend Lead:** ********\_\_\_\_********  
  **Date:** ********\_\_\_\_********

- **Security/Privacy Officer:** ********\_\_\_\_********  
  **Date:** ********\_\_\_\_********

- **QA Lead:** ********\_\_\_\_********  
  **Date:** ********\_\_\_\_********

Notes / Known Deviations:

- ***
- ***
- ***
