# Authentication Patterns Reference

Detailed implementation patterns for authentication and session management. Covers password hashing, JWT, refresh token rotation, RBAC middleware, timing-safe comparison, and session management.

---

## Password Hashing: bcrypt and argon2id

### bcrypt

```typescript
import bcrypt from 'bcrypt'

const SALT_ROUNDS = 12  // minimum for production; increase if hardware allows

// Hashing (during registration or password change)
export async function hashPassword(plainPassword: string): Promise<string> {
  return bcrypt.hash(plainPassword, SALT_ROUNDS)
}

// Verification (during login)
export async function verifyPassword(candidate: string, hash: string): Promise<boolean> {
  return bcrypt.compare(candidate, hash)
}
```

### argon2id (preferred for new systems — OWASP recommendation)

```typescript
import argon2 from 'argon2'

// Hashing
export async function hashPassword(plainPassword: string): Promise<string> {
  return argon2.hash(plainPassword, {
    type: argon2.argon2id,
    memoryCost: 65536,  // 64 MB
    timeCost: 3,        // iterations
    parallelism: 4,
  })
}

// Verification
export async function verifyPassword(candidate: string, hash: string): Promise<boolean> {
  return argon2.verify(hash, candidate)
}
```

**Never use MD5, SHA-1, SHA-256, or unsalted hashes for passwords.** These are fast hashing algorithms designed for data integrity, not password storage. Detect them:

```
createHash\(['"]md5['"]|createHash\(['"]sha1['"]|createHash\(['"]sha256['"]
```

---

## JWT: Sign, Verify, and Storage

```typescript
import jwt from 'jsonwebtoken'

const JWT_SECRET = process.env.JWT_SECRET!
const JWT_ALGORITHM = 'HS256' as const

// Sign — keep access tokens short-lived (15 minutes)
export function signAccessToken(payload: { sub: string; role: string }): string {
  return jwt.sign(payload, JWT_SECRET, {
    algorithm: JWT_ALGORITHM,
    expiresIn: '15m',
  })
}

// Verify — always specify the algorithms array to prevent "alg: none" attack
export function verifyAccessToken(token: string): jwt.JwtPayload {
  return jwt.verify(token, JWT_SECRET, {
    algorithms: [JWT_ALGORITHM],  // never leave this open-ended or omit
  }) as jwt.JwtPayload
}

// Store in httpOnly cookie — NOT localStorage (vulnerable to XSS)
export function setAccessTokenCookie(res: Response, token: string): void {
  res.cookie('access_token', token, {
    httpOnly: true,                                          // inaccessible to JS
    secure: process.env.NODE_ENV === 'production',           // HTTPS only in prod
    sameSite: 'strict',                                      // CSRF protection
    maxAge: 15 * 60 * 1000,                                  // 15 min in ms
  })
}
```

---

## Refresh Token Rotation

Refresh tokens are long-lived but must be rotated on each use. Reuse of an old refresh token signals theft — invalidate the entire family.

```typescript
import { randomBytes } from 'crypto'

// Generate a cryptographically random refresh token
export function generateRefreshToken(): string {
  return randomBytes(64).toString('hex')
}

// Store hashed in DB — never store plain refresh tokens
export async function saveRefreshToken(userId: string, token: string): Promise<void> {
  const hashed = createHash('sha256').update(token).digest('hex')
  await db.refreshTokens.create({
    data: {
      userId,
      tokenHash: hashed,
      expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000), // 30 days
    },
  })
}

// Rotate: validate old token, issue new token, invalidate old
export async function rotateRefreshToken(
  oldToken: string
): Promise<{ accessToken: string; refreshToken: string }> {
  const hashed = createHash('sha256').update(oldToken).digest('hex')
  const record = await db.refreshTokens.findFirst({ where: { tokenHash: hashed } })

  if (!record || record.expiresAt < new Date()) {
    // Token not found or expired — possible replay attack
    // Invalidate all refresh tokens for safety if you track token families
    throw new Error('Invalid or expired refresh token')
  }

  // Delete old token — single use
  await db.refreshTokens.delete({ where: { id: record.id } })

  const user = await db.users.findUniqueOrThrow({ where: { id: record.userId } })
  const accessToken = signAccessToken({ sub: user.id, role: user.role })
  const refreshToken = generateRefreshToken()
  await saveRefreshToken(user.id, refreshToken)

  return { accessToken, refreshToken }
}
```

---

## RBAC Middleware

```typescript
import { Request, Response, NextFunction } from 'express'

// Authenticate: verify JWT and attach user to request
export function authenticate(req: Request, res: Response, next: NextFunction): void {
  const token = req.cookies?.access_token
  if (!token) {
    res.status(401).json({ error: 'Unauthorized' })
    return
  }
  try {
    req.user = verifyAccessToken(token)
    next()
  } catch {
    res.status(401).json({ error: 'Invalid or expired token' })
  }
}

// Authorize: require specific roles
export function requireRole(...roles: string[]) {
  return (req: Request, res: Response, next: NextFunction): void => {
    if (!req.user || !roles.includes(req.user.role)) {
      res.status(403).json({ error: 'Forbidden' })
      return
    }
    next()
  }
}

// Usage — compose on routes
router.get('/profile',            authenticate,                         getProfile)
router.patch('/profile',          authenticate,                         updateProfile)
router.get('/admin/users',        authenticate, requireRole('admin'),   listUsers)
router.delete('/admin/users/:id', authenticate, requireRole('admin'),   deleteUser)
```

**ABAC (Attribute-Based) pattern for ownership checks:**

```typescript
export function requireOwnership(
  getResourceUserId: (req: Request) => Promise<string | null>
) {
  return async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    const resourceUserId = await getResourceUserId(req)
    if (resourceUserId !== req.user?.sub && req.user?.role !== 'admin') {
      res.status(403).json({ error: 'Forbidden' })
      return
    }
    next()
  }
}

// Usage
router.put(
  '/orders/:id',
  authenticate,
  requireOwnership(async req => {
    const order = await db.orders.findUnique({ where: { id: req.params.id } })
    return order?.userId ?? null
  }),
  updateOrder
)
```

---

## Timing-Safe Comparison

Use `crypto.timingSafeEqual` for any secret comparison to prevent timing side-channel attacks.

```typescript
import { timingSafeEqual, createHash } from 'crypto'

// Hash both values to equal-length buffers before comparing
// (timingSafeEqual requires same-length buffers)
export function safeCompare(a: string, b: string): boolean {
  const bufA = createHash('sha256').update(a).digest()
  const bufB = createHash('sha256').update(b).digest()
  return timingSafeEqual(bufA, bufB)
}

// Use for webhook secrets, API key comparison, magic link tokens
export function verifyWebhookSecret(received: string, expected: string): boolean {
  return safeCompare(received, expected)
}
```

**Never use `===` for secret comparison** — it short-circuits on the first differing character and leaks timing information about how many characters matched.

---

## Session Management

### Account lockout after repeated failures

```typescript
const MAX_ATTEMPTS = 5
const LOCKOUT_WINDOW_MS = 15 * 60 * 1000  // 15 minutes

export async function recordFailedLogin(userId: string): Promise<void> {
  const key = `login:attempts:${userId}`
  const attempts = await redis.incr(key)
  if (attempts === 1) {
    await redis.expire(key, LOCKOUT_WINDOW_MS / 1000)
  }
  if (attempts >= MAX_ATTEMPTS) {
    await redis.set(`login:locked:${userId}`, '1', 'EX', LOCKOUT_WINDOW_MS / 1000)
    logger.warn('auth.account_locked', { userId })
  }
}

export async function isAccountLocked(userId: string): Promise<boolean> {
  return (await redis.exists(`login:locked:${userId}`)) === 1
}
```

### Session ID rotation on privilege elevation

```typescript
// Express-session: regenerate session after login to prevent session fixation
export async function login(req: Request, res: Response): Promise<void> {
  const user = await authenticateUser(req.body.email, req.body.password)
  if (!user) {
    res.status(401).json({ error: 'Invalid credentials' })
    return
  }

  // Regenerate session ID — prevents session fixation attacks
  await new Promise<void>((resolve, reject) =>
    req.session.regenerate(err => (err ? reject(err) : resolve()))
  )

  req.session.userId = user.id
  req.session.role = user.role
  res.json({ message: 'Logged in' })
}
```

### Secure logout: invalidate server-side

```typescript
// For stateless JWTs: maintain a short-lived denylist in Redis
export async function revokeToken(jti: string, expiresAt: number): Promise<void> {
  const ttl = Math.max(0, expiresAt - Math.floor(Date.now() / 1000))
  await redis.set(`jwt:revoked:${jti}`, '1', 'EX', ttl)
}

export async function isTokenRevoked(jti: string): Promise<boolean> {
  return (await redis.exists(`jwt:revoked:${jti}`)) === 1
}

// Add jti claim when signing tokens
const token = jwt.sign(
  { sub: user.id, role: user.role, jti: randomUUID() },
  JWT_SECRET,
  { algorithm: 'HS256', expiresIn: '15m' }
)
```

### Session management checklist

- [ ] Rotate session ID on login (prevent session fixation)
- [ ] Invalidate session/token server-side on logout
- [ ] Refresh tokens are single-use and rotated on each use
- [ ] Account lockout triggers after 5 failed attempts within 15 minutes
- [ ] All secret comparisons use `crypto.timingSafeEqual`
- [ ] Re-authentication required for sensitive operations (email/password change, payment method update)
- [ ] Session tokens stored in `httpOnly`, `Secure`, `SameSite=Strict` cookies
