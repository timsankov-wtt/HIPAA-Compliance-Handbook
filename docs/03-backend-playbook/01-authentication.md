# Authentication & Authorization

> Secure user authentication and role-based access control for HIPAA compliance

---

## Overview

Authentication and authorization are **critical technical safeguards** under HIPAA Security Rule § 164.312(a) and § 164.312(d). Every user accessing ePHI must be uniquely identified, authenticated, and authorized based on their role and job function.

As of the 2025 HIPAA Security Rule updates, **Multi-Factor Authentication (MFA) is now mandatory** for all access to systems containing ePHI—not just remote access.

---

## HIPAA Requirements

### Access Control (§ 164.312(a)(1))

**Required Implementation Specifications:**

1. **Unique User Identification (§ 164.312(a)(2)(i))**
   - Assign a unique name and/or number for identifying and tracking user identity
   - No shared accounts or generic logins

2. **Emergency Access Procedure (§ 164.312(a)(2)(ii))**
   - Establish procedures for obtaining necessary ePHI during an emergency

3. **Automatic Logoff (§ 164.312(a)(2)(iii))**
   - Implement electronic procedures that terminate an electronic session after a predetermined time of inactivity

4. **Encryption and Decryption (§ 164.312(a)(2)(iv))**
   - Implement mechanisms to encrypt and decrypt ePHI (now mandatory as of 2025)

### Person or Entity Authentication (§ 164.312(d))

**Required:** Implement procedures to verify that a person or entity seeking access to ePHI is the one claimed.

**2025 Update:** Multi-Factor Authentication (MFA) is now **required** (no longer "addressable") for all access to systems containing ePHI.

---

## Multi-Factor Authentication (MFA)

### 2025 Mandate

The January 2025 HIPAA Security Rule NPRM makes MFA **mandatory** across all access points to ePHI:

- **Previously:** MFA was "addressable" (recommended but optional with risk analysis)
- **Now:** MFA is **required** for both remote AND internal access to all systems that create, receive, maintain, or transmit ePHI
- **Timeline:** Final rule expected late 2025/early 2026, with ~12 months for implementation

### MFA Factors

Authentication must use at least **two of three** factor types:

1. **Something You Know** - Password, PIN, security question
2. **Something You Have** - Mobile device, hardware token, smart card
3. **Something You Are** - Biometric (fingerprint, face recognition, behavioral biometrics)

### Implementation Options

**TOTP (Time-Based One-Time Password)** ✅ Recommended
- Apps: Google Authenticator, Authy, Microsoft Authenticator
- Industry standard, widely supported
- Works offline
- Low cost

```javascript
/**
 * Enable TOTP MFA for user
 * @param userId - Unique user identifier
 * @returns QR code and secret for user to scan
 */
async enableTOTP(userId: string) {
  const secret = speakeasy.generateSecret({ name: 'Healthcare App' });

  // Store secret encrypted in database
  await this.userService.updateMFASecret(userId, secret.base32);

  // Return QR code for user to scan
  return {
    qrCode: secret.otpauth_url,
    secret: secret.base32
  };
}

/**
 * Verify TOTP token during login
 */
async verifyTOTP(userId: string, token: string): Promise<boolean> {
  const user = await this.userService.findById(userId);

  return speakeasy.totp.verify({
    secret: user.mfaSecret,
    encoding: 'base32',
    token: token,
    window: 2 // Allow 2 time steps of drift
  });
}
```

**SMS/Email Codes** ⚠️ Use with Caution
- Vulnerable to SIM swapping and phishing
- HIPAA-compliant provider required (Twilio expensive ~$10K+)
- Not recommended as sole MFA method

**Hardware Tokens (FIDO2/WebAuthn)** ✅ Most Secure
- YubiKey, hardware security keys
- Phishing-resistant
- Higher upfront cost
- Best for high-privilege accounts

**Biometric Authentication** ✅ Approved
- 2025 NPRM recognizes behavioral biometrics
- Fingerprint, face recognition
- Often device-dependent (mobile apps)

**Backup Codes**
Always provide recovery codes in case primary MFA method fails:

```javascript
/**
 * Generate backup recovery codes
 */
generateBackupCodes(userId: string): string[] {
  const codes = [];
  for (let i = 0; i < 10; i++) {
    // Generate cryptographically secure random codes
    codes.push(crypto.randomBytes(4).toString('hex').toUpperCase());
  }

  // Store hashed versions in database
  await this.userService.saveBackupCodes(userId, codes);

  return codes; // Show once to user
}
```

---

## Password Policies

### HIPAA Requirements

HIPAA does not mandate specific password requirements but refers to **NIST Special Publication 800-63B** as the standard:

### Minimum Requirements

**Password Length:**
- **Minimum:** 8 characters (NIST baseline)
- **Recommended:** 12+ characters for stronger security
- **Maximum:** Allow up to 64 characters for passphrases

**Password Complexity:**
- Mix of uppercase, lowercase, numbers, and special characters
- **Important:** NIST now emphasizes **length over complexity**
- Encourage passphrases (e.g., "CorrectHorseBatteryStaple") over complex short passwords

**What to Avoid:**
- ❌ Dictionary words
- ❌ Common passwords (123456, password, admin)
- ❌ User's name or username
- ❌ Password hints that reveal the password
- ❌ Previously breached passwords

### Password Storage

**Never store passwords in plain text.** Always hash with a strong algorithm:

```javascript
import * as bcrypt from 'bcrypt';

/**
 * Hash password with bcrypt
 * HIPAA Best Practice: Use bcrypt with cost factor 12+
 */
async hashPassword(password: string): Promise<string> {
  const saltRounds = 12; // Higher = more secure but slower
  return await bcrypt.hash(password, saltRounds);
}

/**
 * Verify password during login
 */
async verifyPassword(password: string, hash: string): Promise<boolean> {
  return await bcrypt.compare(password, hash);
}
```

**Best Practices:**
- Use **bcrypt**, **argon2**, or **PBKDF2** (not MD5 or SHA-1!)
- Salt rounds: Minimum 10, recommended 12+
- Never log passwords, even in error messages
- Implement rate limiting on login attempts

### Password Change Policy

**NIST Current Guidance:**
- ❌ **Do NOT** require periodic password changes (e.g., every 90 days)
- ✅ **Do** require password change when:
  - Evidence of compromise
  - User reports suspicious activity
  - Employee termination or role change
  - Initial temporary password must be changed on first login

**Password Validation Example:**

```javascript
/**
 * Validate password meets HIPAA-aligned NIST standards
 */
validatePassword(password: string, username: string): ValidationResult {
  const errors = [];

  // Minimum length
  if (password.length < 12) {
    errors.push('Password must be at least 12 characters');
  }

  // Check against common passwords
  if (COMMON_PASSWORDS.includes(password.toLowerCase())) {
    errors.push('This password is too common');
  }

  // No username in password
  if (password.toLowerCase().includes(username.toLowerCase())) {
    errors.push('Password cannot contain username');
  }

  // Complexity check (optional but recommended)
  const hasUpper = /[A-Z]/.test(password);
  const hasLower = /[a-z]/.test(password);
  const hasNumber = /[0-9]/.test(password);
  const hasSpecial = /[!@#$%^&*(),.?":{}|<>]/.test(password);

  const complexityCount = [hasUpper, hasLower, hasNumber, hasSpecial]
    .filter(Boolean).length;

  if (complexityCount < 3) {
    errors.push('Use mix of uppercase, lowercase, numbers, and symbols');
  }

  return {
    valid: errors.length === 0,
    errors
  };
}
```

---

## Role-Based Access Control (RBAC)

### HIPAA Minimum Necessary Standard

**Principle:** Users should only access the minimum PHI necessary to perform their job function.

### Designing Roles

**Common Healthcare Roles:**

```typescript
enum Role {
  PATIENT = 'patient',               // Access own records only
  PROVIDER = 'provider',              // Access assigned patients
  NURSE = 'nurse',                    // Access patients in care
  ADMIN = 'admin',                    // Administrative access
  BILLING = 'billing',                // Billing information only
  SUPPORT = 'support',                // Limited support access
  SUPER_ADMIN = 'super_admin'         // Full system access
}

enum Permission {
  PHI_READ = 'phi:read',
  PHI_WRITE = 'phi:write',
  PHI_DELETE = 'phi:delete',
  PATIENT_CREATE = 'patient:create',
  APPOINTMENT_MANAGE = 'appointment:manage',
  BILLING_VIEW = 'billing:view',
  AUDIT_LOG_VIEW = 'audit:view',
  USER_MANAGE = 'user:manage'
}
```

**Role-Permission Mapping:**

```typescript
const ROLE_PERMISSIONS = {
  [Role.PATIENT]: [
    Permission.PHI_READ  // Own records only
  ],
  [Role.PROVIDER]: [
    Permission.PHI_READ,
    Permission.PHI_WRITE,
    Permission.APPOINTMENT_MANAGE
  ],
  [Role.NURSE]: [
    Permission.PHI_READ,
    Permission.PHI_WRITE
  ],
  [Role.BILLING]: [
    Permission.BILLING_VIEW,
    Permission.PHI_READ  // Limited to billing info
  ],
  [Role.ADMIN]: [
    Permission.USER_MANAGE,
    Permission.AUDIT_LOG_VIEW
  ],
  [Role.SUPER_ADMIN]: Object.values(Permission)
};
```

### NestJS RBAC Implementation

**Guard for Role-Based Access:**

```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

/**
 * RBAC Guard - Enforces role-based access control
 * Used as @UseGuards(RolesGuard) on controllers/routes
 */
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    // Get required roles from @Roles() decorator
    const requiredRoles = this.reflector.get<Role[]>(
      'roles',
      context.getHandler()
    );

    if (!requiredRoles) {
      return true; // No roles required
    }

    const request = context.switchToHttp().getRequest();
    const user = request.user;

    if (!user) {
      return false; // Not authenticated
    }

    // Check if user has any of the required roles
    const hasRole = requiredRoles.some(role => user.roles?.includes(role));

    if (!hasRole) {
      // Log unauthorized access attempt
      this.auditLog.logAccessDenied(user.id, request.path);
    }

    return hasRole;
  }
}
```

**Custom Decorator:**

```typescript
import { SetMetadata } from '@nestjs/common';

/**
 * Roles decorator - Specify required roles for endpoint
 * Usage: @Roles(Role.PROVIDER, Role.NURSE)
 */
export const Roles = (...roles: Role[]) => SetMetadata('roles', roles);
```

**Controller Usage:**

```typescript
@Controller('patients')
@UseGuards(JwtAuthGuard, RolesGuard)
export class PatientsController {

  /**
   * Get patient details
   * Only providers and nurses can access
   */
  @Get(':id')
  @Roles(Role.PROVIDER, Role.NURSE)
  async getPatient(@Param('id') id: string, @User() user: UserEntity) {
    // Additional check: Can this user access THIS patient?
    if (!await this.canAccessPatient(user.id, id)) {
      throw new ForbiddenException('Cannot access this patient');
    }

    // Log PHI access
    await this.auditLog.log({
      userId: user.id,
      action: 'PHI_READ',
      resource: 'patient',
      resourceId: id,
      timestamp: new Date()
    });

    return this.patientsService.findOne(id);
  }

  /**
   * Delete patient record
   * Only providers can delete (with hard delete requirement!)
   */
  @Delete(':id')
  @Roles(Role.PROVIDER, Role.SUPER_ADMIN)
  async deletePatient(@Param('id') id: string, @User() user: UserEntity) {
    // Log before deletion
    await this.auditLog.log({
      userId: user.id,
      action: 'PHI_DELETE',
      resource: 'patient',
      resourceId: id,
      timestamp: new Date()
    });

    // Hard delete - no soft deletes allowed for PHI!
    return this.patientsService.remove(id);
  }
}
```

---

## Session Management

### HIPAA Requirements

**Automatic Logoff (§ 164.312(a)(2)(iii)):**
Sessions must terminate after a predetermined period of inactivity.

### Best Practices

**Session Timeout:**
- **Recommended:** 15-30 minutes of inactivity
- **Maximum:** No more than 60 minutes for systems with PHI
- **High-risk areas:** 5-10 minutes (e.g., billing, admin panels)

**Implementation with JWT:**

```typescript
/**
 * Generate JWT with expiration
 */
generateToken(user: UserEntity): string {
  const payload = {
    sub: user.id,
    username: user.username,
    roles: user.roles,
    mfaVerified: true  // Only set after MFA verification
  };

  return this.jwtService.sign(payload, {
    expiresIn: '30m'  // 30-minute expiration
  });
}

/**
 * Refresh token strategy
 */
generateRefreshToken(user: UserEntity): string {
  return this.jwtService.sign(
    { sub: user.id, type: 'refresh' },
    {
      expiresIn: '7d',  // Longer expiration for refresh
      secret: process.env.REFRESH_TOKEN_SECRET
    }
  );
}
```

**Session Storage:**
- **Access tokens:** Short-lived (15-30 min), stored in memory or httpOnly cookies
- **Refresh tokens:** Longer-lived (7 days max), stored in httpOnly, secure cookies
- **Never:** Store tokens in localStorage (XSS vulnerability)

```typescript
/**
 * Set secure cookie with refresh token
 */
setRefreshTokenCookie(res: Response, token: string) {
  res.cookie('refreshToken', token, {
    httpOnly: true,      // Not accessible via JavaScript
    secure: true,        // HTTPS only
    sameSite: 'strict',  // CSRF protection
    maxAge: 7 * 24 * 60 * 60 * 1000  // 7 days
  });
}
```

### Activity Tracking

Track last activity to enforce timeout:

```typescript
/**
 * Middleware to update last activity timestamp
 */
@Injectable()
export class ActivityMiddleware implements NestMiddleware {
  async use(req: Request, res: Response, next: Function) {
    if (req.user) {
      // Update last activity in Redis/cache
      await this.cacheService.set(
        `user:${req.user.id}:lastActivity`,
        Date.now(),
        { ttl: 1800 }  // 30 min TTL
      );
    }
    next();
  }
}
```

---

## JWT Authentication Strategy

### NestJS Passport JWT

```typescript
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';

/**
 * JWT Strategy for validating access tokens
 */
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(private usersService: UsersService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,  // Enforce expiration
      secretOrKey: process.env.JWT_SECRET
    });
  }

  /**
   * Validate JWT payload and attach user to request
   */
  async validate(payload: any) {
    // Verify MFA was completed for this session
    if (!payload.mfaVerified) {
      throw new UnauthorizedException('MFA required');
    }

    // Fetch current user data (in case roles changed)
    const user = await this.usersService.findById(payload.sub);

    if (!user || !user.isActive) {
      throw new UnauthorizedException('User not found or inactive');
    }

    return {
      id: user.id,
      username: user.username,
      roles: user.roles
    };
  }
}
```

### Two-Step Authentication Flow

```typescript
/**
 * Step 1: Verify username/password
 */
@Post('login')
async login(@Body() loginDto: LoginDto) {
  const user = await this.authService.validateUser(
    loginDto.username,
    loginDto.password
  );

  if (!user) {
    throw new UnauthorizedException('Invalid credentials');
  }

  // Check if MFA is enabled
  if (user.mfaEnabled) {
    // Return temporary token for MFA step
    const mfaToken = this.authService.generateMFAToken(user.id);
    return {
      requiresMFA: true,
      mfaToken  // Valid for 5 minutes only
    };
  }

  // No MFA (should not happen in HIPAA systems after 2025!)
  return this.authService.generateTokens(user);
}

/**
 * Step 2: Verify MFA token
 */
@Post('mfa/verify')
async verifyMFA(@Body() mfaDto: MFADto) {
  // Decode temporary MFA token
  const mfaPayload = await this.authService.verifyMFAToken(mfaDto.mfaToken);

  // Verify TOTP code
  const isValid = await this.authService.verifyTOTP(
    mfaPayload.userId,
    mfaDto.code
  );

  if (!isValid) {
    throw new UnauthorizedException('Invalid MFA code');
  }

  // MFA successful - issue full access token
  const user = await this.usersService.findById(mfaPayload.userId);

  // Log successful authentication
  await this.auditLog.log({
    userId: user.id,
    action: 'USER_LOGIN',
    details: { mfaUsed: true },
    timestamp: new Date()
  });

  return this.authService.generateTokens(user);
}
```

---

## Database Schema Considerations

### Users Table

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  username VARCHAR(255) UNIQUE NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,

  -- MFA fields
  mfa_enabled BOOLEAN DEFAULT false,
  mfa_secret TEXT,  -- Encrypted TOTP secret
  backup_codes TEXT[],  -- Hashed backup codes

  -- Account status
  is_active BOOLEAN DEFAULT true,
  is_locked BOOLEAN DEFAULT false,
  failed_login_attempts INT DEFAULT 0,
  last_login TIMESTAMP,

  -- Audit fields
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  created_by UUID REFERENCES users(id),
  updated_by UUID REFERENCES users(id)
);

CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
```

### Roles & Permissions

```sql
CREATE TABLE roles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(50) UNIQUE NOT NULL,
  description TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE permissions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) UNIQUE NOT NULL,
  resource VARCHAR(50) NOT NULL,
  action VARCHAR(50) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE role_permissions (
  role_id UUID REFERENCES roles(id) ON DELETE CASCADE,
  permission_id UUID REFERENCES permissions(id) ON DELETE CASCADE,
  PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_roles (
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  role_id UUID REFERENCES roles(id) ON DELETE CASCADE,
  assigned_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  assigned_by UUID REFERENCES users(id),
  PRIMARY KEY (user_id, role_id)
);
```

### Prisma Schema Example

```prisma
model User {
  id            String   @id @default(uuid())
  username      String   @unique
  email         String   @unique
  passwordHash  String   @map("password_hash")

  // MFA
  mfaEnabled    Boolean  @default(false) @map("mfa_enabled")
  mfaSecret     String?  @map("mfa_secret")
  backupCodes   String[] @map("backup_codes")

  // Status
  isActive      Boolean  @default(true) @map("is_active")
  isLocked      Boolean  @default(false) @map("is_locked")
  failedLogins  Int      @default(0) @map("failed_login_attempts")
  lastLogin     DateTime? @map("last_login")

  // Relations
  roles         UserRole[]
  auditLogs     AuditLog[]

  createdAt     DateTime @default(now()) @map("created_at")
  updatedAt     DateTime @updatedAt @map("updated_at")

  @@map("users")
}

model Role {
  id          String   @id @default(uuid())
  name        String   @unique
  description String?

  users       UserRole[]
  permissions RolePermission[]

  createdAt   DateTime @default(now()) @map("created_at")

  @@map("roles")
}

model UserRole {
  userId     String   @map("user_id")
  roleId     String   @map("role_id")
  assignedAt DateTime @default(now()) @map("assigned_at")

  user       User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  role       Role     @relation(fields: [roleId], references: [id], onDelete: Cascade)

  @@id([userId, roleId])
  @@map("user_roles")
}
```

---

## Common Mistakes & How to Avoid Them

### ❌ Mistake 1: Shared Accounts
**Problem:** Multiple users sharing one login (e.g., "admin", "nurse_station")
**HIPAA Violation:** § 164.312(a)(2)(i) Unique User Identification
**Solution:** Each person must have their own unique account

### ❌ Mistake 2: No MFA
**Problem:** Only password authentication
**HIPAA Violation:** 2025 mandate requires MFA
**Solution:** Implement TOTP or hardware token MFA immediately

### ❌ Mistake 3: Weak Password Storage
**Problem:** Storing passwords with MD5 or plain text
**HIPAA Violation:** § 164.312(a)(2)(iv) Encryption
**Solution:** Use bcrypt with salt rounds 12+

### ❌ Mistake 4: No Session Timeout
**Problem:** Sessions never expire
**HIPAA Violation:** § 164.312(a)(2)(iii) Automatic Logoff
**Solution:** 30-minute JWT expiration, enforce on backend

### ❌ Mistake 5: Role Checks Only in Frontend
**Problem:** Authorization only validated in React/UI
**HIPAA Violation:** Can be bypassed with API calls
**Solution:** Always enforce RBAC on backend with Guards

### ❌ Mistake 6: Not Logging Auth Events
**Problem:** No audit trail of login attempts
**HIPAA Violation:** § 164.312(b) Audit Controls
**Solution:** Log all authentication events (success and failure)

### ❌ Mistake 7: Storing Tokens in localStorage
**Problem:** Tokens accessible via JavaScript (XSS vulnerability)
**Security Risk:** High
**Solution:** Use httpOnly, secure cookies for refresh tokens

---

## Testing Authentication

### Unit Tests

```typescript
describe('AuthService', () => {
  it('should hash passwords with bcrypt', async () => {
    const password = 'SecurePass123!';
    const hash = await authService.hashPassword(password);

    expect(hash).not.toEqual(password);
    expect(await bcrypt.compare(password, hash)).toBe(true);
  });

  it('should enforce MFA for all users', async () => {
    const user = await usersService.create({
      username: 'testuser',
      password: 'SecurePass123!'
    });

    expect(user.mfaEnabled).toBe(true);  // Should be mandatory
  });

  it('should expire JWT after 30 minutes', () => {
    const token = authService.generateToken(user);
    const decoded = jwtService.decode(token);

    const expiresIn = decoded.exp - decoded.iat;
    expect(expiresIn).toBeLessThanOrEqual(1800);  // 30 min = 1800 sec
  });
});
```

### Integration Tests

```typescript
describe('Authentication E2E', () => {
  it('should require MFA after password verification', async () => {
    const response = await request(app.getHttpServer())
      .post('/auth/login')
      .send({ username: 'testuser', password: 'SecurePass123!' });

    expect(response.body.requiresMFA).toBe(true);
    expect(response.body.mfaToken).toBeDefined();
  });

  it('should reject requests without valid JWT', async () => {
    const response = await request(app.getHttpServer())
      .get('/patients/123')
      .expect(401);
  });

  it('should enforce role-based access', async () => {
    const billingUserToken = await getToken('billing_user');

    const response = await request(app.getHttpServer())
      .delete('/patients/123')  // Only providers can delete
      .set('Authorization', `Bearer ${billingUserToken}`)
      .expect(403);
  });
});
```

---

## Checklist

Before deploying authentication system:

- [ ] **Unique user identification** - No shared accounts
- [ ] **MFA implemented** - TOTP or hardware tokens
- [ ] **Password policy enforced** - 12+ characters, complexity
- [ ] **Bcrypt password hashing** - Salt rounds 12+
- [ ] **JWT short expiration** - 30 minutes or less
- [ ] **Refresh token strategy** - Stored in httpOnly cookies
- [ ] **RBAC implemented** - Guards on all endpoints
- [ ] **Session timeout** - Auto-logout after inactivity
- [ ] **Rate limiting** - Prevent brute force attacks
- [ ] **Account lockout** - After failed login attempts
- [ ] **Audit logging** - All auth events logged
- [ ] **Emergency access procedure** - Documented and tested
- [ ] **MFA backup codes** - Provided to users
- [ ] **No tokens in localStorage** - Use secure cookies
- [ ] **Role assignment audited** - Log who assigns roles

---

## Additional Resources

**HIPAA Guidance:**
- [HHS Security Rule § 164.312](https://www.hhs.gov/hipaa/for-professionals/security/laws-regulations/index.html)
- [2025 HIPAA MFA Requirements](https://www.strongdm.com/blog/hipaa-mfa-requirements)

**NIST Standards:**
- [NIST 800-63B - Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [NIST Password Guidelines](https://www.hipaasecurenow.com/nist-guidelines-for-strong-passwords/)

**Implementation:**
- [NestJS Authentication Documentation](https://docs.nestjs.com/security/authentication)
- [Passport.js Strategies](http://www.passportjs.org/)

---

## Next Steps

Continue to the next section:

**[→ Encryption](02-encryption.md)**

Or explore related topics:
- [Audit Logging](03-audit-logging.md) - Log authentication events
- [Backend Checklist](05-checklist.md) - Pre-deployment verification

---

*Last Updated: November 2025*
*Part of the HIPAA Compliance Handbook - Backend Playbook*
