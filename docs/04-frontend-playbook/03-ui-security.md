# UI Security & Privacy UX

> Designing user interfaces that protect PHI while staying usable

---

## 1. Data Masking in the UI

Not all users should see all PHI, all the time. Masking reduces accidental exposure while still supporting workflows.

### 1.1 When to Mask

Mask sensitive data by default, especially:

- National identifiers (e.g., SSN)
- Full dates of birth
- Contact information (phone, email)
- Financial or insurance details

Example patterns:

- `***-**-1234` instead of full SSN
- `**** **** **** 1234` for card-like numbers (if present)
- `1970‑04‑**` for dates of birth (only day hidden)

### 1.2 Show/Hide Toggles

Allow authorized users to reveal full values **intentionally**:

- Require explicit click on a “Show” icon/button
- Optionally time‑limit visibility (auto‑hide after a few seconds)
- Log sensitive reveals on the backend where appropriate (e.g., viewing full SSN)

_Textual mockup:_

> Patient Details Card
>
> - Name: Jane D.
> - DOB: 1985‑03‑\*\* [Show]
> - Phone: (**_) _**‑1234 [Show]
> - Diagnosis: Chronic condition (general) [View full notes]

---

## 2. Role‑Based UI Rendering

Authorization is primarily a **backend** concern, but the UI must:

- Avoid showing controls or data that the current role should never use
- Guard sensitive components with role checks

### 2.1 Conditional Rendering in React

```tsx
function PatientDetails({ patient, user }) {
  const canViewFullPHI =
    user.roles.includes("provider") || user.roles.includes("admin");

  return (
    <div>
      <h2>{patient.displayName}</h2>
      <p>
        Date of birth: {canViewFullPHI ? patient.dateOfBirth : "****-**-**"}
      </p>
      {canViewFullPHI && <button>View full medical history</button>}
    </div>
  );
}
```

This does **not** replace backend checks; APIs must still enforce access control.

### 2.2 Access Control Patterns

- Use React Context or a dedicated auth provider to expose current user roles/permissions.
- Build Higher‑Order Components (HOCs) or hooks (`useCan`) to centralize role logic.
- In Flutter, use state management (Provider, Riverpod, Bloc, etc.) to propagate permissions and hide features accordingly.

---

## 3. Error Handling (No PHI in Errors)

Error messages are often copied into tickets, logs, or screenshots. They must not contain PHI.

### 3.1 User‑Facing Errors

- Be **generic and helpful**, not specific about patient details.

Examples:

- ✅ `"We couldn’t load this patient record. Please try again or contact support."`
- ❌ `"Patient John Doe with SSN 123‑45‑6789 not found in database."`

### 3.2 Logging Errors

- Do **not** log form values or full responses that may contain PHI.
- Aggregate by IDs, not names.

```tsx
// ❌ Avoid
console.error("Failed to save form", formValues); // Likely contains PHI

// ✅ Better
console.error("Failed to save patient form", { patientId: patient.id, error });
```

---

## 4. Session Management in the UI

Frontend must respect backend session and token policies.

### 4.1 Auto‑Logout on Inactivity

- Implement idle timeout detection in SPA/mobile apps.
- Show a **warning dialog** before logging out.
- After timeout, clear any local state and redirect to login.

_Example flow:_

1. No user interaction for N minutes.
2. Show: “You will be signed out in 2 minutes due to inactivity. Stay signed in?”
3. If no response → log out and navigate to login.

### 4.2 “Keep Me Logged In”

- Use only when backed by secure token design on the backend.
- Never implement this by storing PHI or long‑lived secrets in localStorage.

---

## 5. Secure Development Practices

### 5.1 No `console.log` of PHI

- Treat console output as **insecure**; it may be recorded in logs, browser extensions, or screenshots.
- Use feature flags to completely disable debug logging in production builds.

### 5.2 DevTools & Source Maps

- Disable or restrict detailed source maps in production if they expose internal structure or comments about PHI handling.
- Ensure no test PHI or real PHI appears in comments or sample data.

### 5.3 Third‑Party Tools

- Carefully review any analytics / error tracking / A/B testing tools.
- Ensure **no PHI** is sent in events, labels, or metadata.
- Use anonymous IDs instead of patient identifiers.

---

## 6. React/Next.js Patterns

### 6.1 Higher‑Order Components for Access Control

```tsx
function withRole(Component: React.ComponentType<any>, allowedRoles: string[]) {
  return function GuardedComponent(props: any) {
    const { user } = useAuth();
    if (!user || !allowedRoles.some((r) => user.roles.includes(r))) {
      return <p>Access denied.</p>;
    }
    return <Component {...props} />;
  };
}

const ProviderDashboard = withRole(Dashboard, ["provider", "admin"]);
```

### 6.2 Permission‑Aware Navigation

- Only show navigation items for features the user can actually access.
- Avoid “teasing” sensitive features to unauthorized users.

---

## 7. Flutter Patterns

### 7.1 State Management and Permissions

- Keep user roles/permissions in a secure app‑wide state (e.g., Provider, Riverpod).
- Widgets should check permissions before rendering sensitive content.

```dart
if (user.canViewFullPhi) {
  return Text(patient.fullName);
} else {
  return Text(patient.initials);
}
```

### 7.2 Navigation Guards

- Use routing logic that checks authentication/authorization before entering PHI screens.
- Redirect unauthorized users to an error page or dashboard.

---

## UI Security Checklist

- [ ] Sensitive fields (SSN, DOB, contact info) are masked by default
- [ ] UI exposes only the minimum necessary PHI for each workflow
- [ ] Role‑based rendering prevents unauthorized users from seeing sensitive UI elements
- [ ] Error messages contain no PHI and are safe to screenshot or email
- [ ] No PHI is ever logged to the browser console or in‑app debug logs
- [ ] Session timeouts and idle logout behavior are implemented and tested
- [ ] Production builds do not expose sensitive debugging information or source maps
- [ ] Third‑party tools (analytics, error tracking) are configured to exclude PHI
