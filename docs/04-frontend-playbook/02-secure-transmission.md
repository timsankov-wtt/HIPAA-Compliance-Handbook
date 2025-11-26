# Secure Transmission

> Keeping PHI safe as it travels between frontend and backend

---

## 1. HTTPS Requirements

### 1.1 Enforce HTTPS Everywhere

For HIPAA, **all traffic carrying PHI must be encrypted in transit**:

- Use **HTTPS** with **TLS 1.2+** for all API calls and pages that can display PHI.
- Disable or redirect plain HTTP to HTTPS at the load balancer / CDN level.
- Never allow mixed content (HTTPS page loading HTTP resources).

**Key practices:**

- Configure HSTS (HTTP Strict Transport Security) so browsers automatically prefer HTTPS.
- Use modern TLS configurations on your CDN / load balancer (e.g., AWS ALB, CloudFront).

### 1.2 Certificate Validation

- Frontends must **not** disable certificate validation.
- For web apps, rely on the browser’s TLS stack.
- For mobile (Flutter), ensure HTTP clients validate certificates and reject self-signed / invalid certs in production.

---

## 2. API Authentication & Token Handling

### 2.1 Token Storage Options

HIPAA does not mandate a specific token format, but it strongly influences **how tokens are stored**:

- **Preferred for web:**

  - `httpOnly`, `secure` cookies for access/refresh tokens
  - Protected by browser against JavaScript (mitigates XSS token theft)

- **Avoid where possible:**
  - `localStorage` or `sessionStorage` for long-lived tokens
  - Storing PHI or tokens in IndexedDB

### 2.2 Example: Next.js Using httpOnly Cookies

```tsx
// Example: login handler (API route) sets httpOnly cookie
import type { NextApiRequest, NextApiResponse } from "next";

export default async function login(req: NextApiRequest, res: NextApiResponse) {
  const { email, password } = req.body;
  // Authenticate via backend service

  const jwt = "signed-jwt-token"; // from auth service

  res.setHeader("Set-Cookie", [
    `accessToken=${jwt}; HttpOnly; Secure; SameSite=Strict; Path=/;`,
  ]);

  res.status(200).json({ ok: true });
}
```

On the client, you **do not** read the cookie directly; you rely on the browser to send it with requests.

### 2.3 Token Lifetime & Session Timeout

- Coordinate with the backend on **token TTLs** and idle timeouts.
- Implement UI flows for:
  - Refreshing tokens before they expire
  - Logging out users after inactivity
  - Forcing re-authentication for sensitive actions (e.g., viewing full records)

---

## 3. What NOT to Do with PHI and Tokens

**Never:**

- Store PHI in `localStorage`, `sessionStorage`, or IndexedDB
- Put PHI in URLs or query parameters
- Include PHI in client-side logs or error messages
- Send PHI over unencrypted HTTP

| Pattern                         | Before (Legacy)                      | After (HIPAA‑aligned)                                   |
| ------------------------------- | ------------------------------------ | ------------------------------------------------------- |
| PHI in query params             | `/patient?name=John&dob=1985-04-23`  | `/patient/12345` (ID only; PHI in secure response body) |
| Tokens in `localStorage`        | `localStorage.setItem("token", ...)` | httpOnly secure cookies                                 |
| PHI in error messages           | "Patient John Doe with SSN ..."      | "Patient record not found" (no PHI)                     |
| Unencrypted HTTP in development | `http://api.local`                   | `https://api.local` with dev TLS                        |

---

## 4. CORS Configuration

CORS is typically configured on the **backend or API gateway**, but frontend choices drive which origins are needed.

Best practices:

- Restrict `Access-Control-Allow-Origin` to **known, trusted domains** (no `*` for authenticated endpoints).
- Only allow `Access-Control-Allow-Credentials: true` when absolutely necessary and controlled.
- Keep `Access-Control-Allow-Methods` and `...-Headers` as tight as possible.

Frontend responsibilities:

- Ensure your apps call APIs **only from approved origins**.
- Use environment variables to distinguish dev/staging/prod API URLs.

---

## 5. Next.js / React Specific Considerations

### 5.1 API Routes & SSR

- Treat **API routes** (`/api/*`) as part of your backend surface:

  - Require authentication for any route touching PHI.
  - Avoid sending PHI in SSR HTML if the page might be cached.

- For **server-side rendering (SSR)**:
  - Do not embed raw PHI into HTML where it might be cached or logged.
  - Prefer fetching PHI client-side after the user is authenticated.

### 5.2 Fetching APIs from the Browser

```typescript
async function fetchAppointments() {
  const res = await fetch("/api/appointments", {
    method: "GET",
    credentials: "include", // send httpOnly cookies
  });

  if (!res.ok) {
    throw new Error("Unable to load appointments");
  }

  return res.json();
}
```

- Always use `https://` URLs in production.
- Use `credentials: "include"` if your auth relies on cookies.

---

## 6. Flutter Specific Considerations

### 6.1 HTTP Client Security

Use `https` endpoints and ensure that the HTTP client validates certificates:

```dart
import 'package:http/http.dart' as http;

Future<void> fetchPatient(String id) async {
  final response = await http.get(Uri.https('api.example.com', '/patients/$id'));

  if (response.statusCode != 200) {
    throw Exception('Failed to load patient');
  }
}
```

Avoid custom HTTP overrides that disable certificate checks in production.

### 6.2 Certificate Pinning (Mobile)

For sensitive mobile apps, consider **certificate pinning**:

- Use packages that support pinning public keys or certificates.
- Regularly plan for certificate rotation to avoid lockouts.

### 6.3 Secure Storage

- Use OS-provided secure storage (e.g., Keychain on iOS, Keystore on Android) for session tokens when needed.
- Never store PHI directly in on-device storage unless business requirements demand it and it is encrypted.

---

## 7. Security Checklist

- [ ] All frontend calls to PHI endpoints use `https://` and validated certificates
- [ ] Tokens are stored in httpOnly cookies (web) or secure OS storage (mobile)
- [ ] No PHI is stored in browser storage (localStorage, sessionStorage, IndexedDB)
- [ ] No PHI appears in query parameters, fragment identifiers, or client logs
- [ ] CORS configuration only allows trusted origins and methods
- [ ] Next.js/React pages do not embed PHI in SSR HTML that might be cached
- [ ] Flutter HTTP clients do not disable certificate validation in production
- [ ] Certificate rotation and (if used) pinning strategies are documented and tested
- [ ] Session expiry and logout flows are implemented and tested end-to-end
