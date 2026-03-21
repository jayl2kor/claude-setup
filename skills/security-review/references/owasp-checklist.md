# OWASP Top 10 (2021) Checklist Reference

Full detection patterns, TypeScript fix examples, and configuration snippets for each OWASP Top 10 category.

---

## A01 — Broken Access Control

**Detection patterns:**
```
# IDOR: object IDs taken directly from user input without ownership check
/users\/\$\{req\.(params|body|query)\./
/findById\(req\.(params|body|query)\./
/where.*id.*=.*req\.(params|body|query)\./

# Missing authorization middleware on route handlers
router\.(get|post|put|patch|delete)\(.*,\s*async\s*\(req,\s*res\)
# ^^^ route handler with no auth middleware argument between path and handler
```

**Fix — always verify ownership:**
```typescript
// BAD: fetches any record by ID from URL param
const order = await db.orders.findById(req.params.id)

// GOOD: scope query to the authenticated user
const order = await db.orders.findOne({
  where: { id: req.params.id, userId: req.user.id },
})
if (!order) return res.status(404).json({ error: 'Not found' })
```

**RBAC middleware pattern:**
```typescript
function requireRole(...roles: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user || !roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Forbidden' })
    }
    next()
  }
}

router.delete('/admin/users/:id', authenticate, requireRole('admin'), deleteUser)
```

**Checklist:**
- [ ] Every route has an `authenticate` middleware before the handler
- [ ] All DB queries include `userId: req.user.id` or equivalent ownership scope
- [ ] Admin-only routes are protected with `requireRole('admin')`
- [ ] File downloads verify ownership before serving

---

## A02 — Cryptographic Failures

**Detection patterns (regex):**
```
# Hardcoded secrets in source
(api[_-]?key|secret|password|token|private[_-]?key)\s*[=:]\s*['"][A-Za-z0-9+/=_\-]{8,}['"]
sk-[a-zA-Z0-9]{32,}          # OpenAI-style keys
ghp_[a-zA-Z0-9]{36}          # GitHub PATs
AKIA[0-9A-Z]{16}             # AWS access key IDs
-----BEGIN (RSA |EC )?PRIVATE KEY-----

# Weak hashing for passwords
md5(|sha1(|sha256(.*password
createHash\(['"]md5['"]|createHash\(['"]sha1['"]

# HTTP instead of HTTPS in URLs
http://(?!localhost|127\.0\.0\.1)
```

**Fix — use bcrypt or argon2 for passwords:**
```typescript
import bcrypt from 'bcrypt'
import argon2 from 'argon2'

// bcrypt — 12 rounds minimum in production
const hash = await bcrypt.hash(plainPassword, 12)
const valid = await bcrypt.compare(candidate, hash)

// argon2id — preferred for new systems (OWASP recommendation)
const hash = await argon2.hash(plainPassword, { type: argon2.argon2id })
const valid = await argon2.verify(hash, candidate)
```

**Fix — secrets belong in environment variables:**
```typescript
// NEVER commit this value; load from process.env
const JWT_SECRET = process.env.JWT_SECRET
if (!JWT_SECRET || JWT_SECRET.length < 32) {
  throw new Error('JWT_SECRET must be set and at least 32 characters')
}
```

---

## A03 — Injection

**SQL injection detection:**
```
# String concatenation in SQL
`SELECT.*\$\{|'.*\+.*req\.|query\(.*\+
# Raw query with user input
db\.query\(`[^`]*\$\{req\.
pool\.query\(`[^`]*\$\{
```

**Fix — parameterized queries only:**
```typescript
// BAD: SQL injection via template literal
const rows = await db.query(`SELECT * FROM users WHERE email = '${email}'`)

// GOOD: parameterized (postgres)
const rows = await db.query('SELECT * FROM users WHERE email = $1', [email])

// GOOD: ORM with type-safe where clause
const user = await prisma.user.findFirst({ where: { email } })
```

**NoSQL injection — MongoDB:**
```typescript
// BAD: object spread from user input
const user = await User.findOne({ ...req.body })

// GOOD: extract only the known field
const user = await User.findOne({ email: String(req.body.email) })
```

**Command injection:**
```typescript
// BAD
const output = execSync(`convert ${req.query.file} output.png`)

// GOOD: pass arguments as an array, never interpolate user input
const { execFileSync } = require('child_process')
const output = execFileSync('convert', [sanitizedFilename, 'output.png'])
```

---

## A04 — Insecure Design

**Checklist:**
- [ ] Threat model exists for the feature (who can abuse it and how?)
- [ ] Rate limiting on all mutating endpoints
- [ ] Sensitive operations require re-authentication (e.g., changing email/password)
- [ ] Business logic limits enforced server-side (amounts, quantities, discounts)
- [ ] Negative testing cases (what happens with invalid/oversized/malformed input?)

---

## A05 — Security Misconfiguration

**Detection patterns:**
```
# Debug mode / stack traces enabled in prod
app\.set\(['"]env['"],\s*['"]development['"]\)
DEBUG\s*=\s*true
NODE_ENV\s*=\s*development

# CORS wildcard
origin:\s*['"]\*['"]|cors\(\)   # cors() with no options = wildcard

# Missing security headers
# (absence of helmet() or equivalent)
```

**Fix — helmet.js for Node.js:**
```typescript
import helmet from 'helmet'

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", 'data:', 'https:'],
      connectSrc: ["'self'", process.env.API_ORIGIN],
    },
  },
  hsts: { maxAge: 63072000, includeSubDomains: true, preload: true },
}))
```

**Fix — CORS with explicit allowlist:**
```typescript
// CORS: explicit allowlist
app.use(cors({
  origin: (origin, callback) => {
    const allowed = (process.env.CORS_ORIGINS ?? '').split(',')
    if (!origin || allowed.includes(origin)) return callback(null, true)
    callback(new Error('CORS not allowed'))
  },
  credentials: true,
}))
```

---

## A06 — Vulnerable and Outdated Components

**Audit commands:**
```bash
# Node.js
npm audit --audit-level=high
npx better-npm-audit audit --level high

# Python
pip-audit

# Continuous scanning in CI
# Add to GitHub Actions:
- uses: actions/dependency-review-action@v4
  with:
    fail-on-severity: high
```

**Supply chain hygiene checklist:**
- [ ] Lock files (`package-lock.json`, `yarn.lock`, `poetry.lock`) committed
- [ ] Use `npm ci` (not `npm install`) in CI — respects lock file exactly
- [ ] Dependabot or Renovate enabled with auto-merge for patch-level updates
- [ ] SBOM generated for each release (`npx @cyclonedx/cyclonedx-npm`)
- [ ] Packages with `>0` direct downloads from typosquatted names audited

---

## A07 — Identification and Authentication Failures

**JWT best practices:**
```typescript
import jwt from 'jsonwebtoken'

// Sign: short expiry, strong algorithm
const token = jwt.sign(
  { sub: user.id, role: user.role },
  process.env.JWT_SECRET!,
  { algorithm: 'HS256', expiresIn: '15m' }  // short-lived access token
)

// Verify: check algorithm explicitly to prevent "alg: none" attack
const payload = jwt.verify(token, process.env.JWT_SECRET!, {
  algorithms: ['HS256'],  // never leave this array open-ended
})

// Store in httpOnly, Secure, SameSite=Strict cookie — NOT localStorage
res.cookie('access_token', token, {
  httpOnly: true,
  secure: process.env.NODE_ENV === 'production',
  sameSite: 'strict',
  maxAge: 15 * 60 * 1000,  // 15 min in ms
})
```

**Session management checklist:**
- [ ] Rotate session ID on privilege elevation (login, sudo)
- [ ] Invalidate session server-side on logout (stateless JWT requires a denylist or short expiry)
- [ ] Implement refresh token rotation — invalidate old refresh token on use
- [ ] Account lockout after N failed login attempts with exponential backoff
- [ ] Timing-safe comparison for secrets (`crypto.timingSafeEqual`)

---

## A08 — Software and Data Integrity Failures

**Checklist:**
- [ ] Verify webhook signatures before processing (Stripe, GitHub, etc.)
- [ ] Use subresource integrity (SRI) for CDN-loaded scripts
- [ ] Pin CI action versions to full SHA digests, not tags

```typescript
// Stripe webhook verification
import Stripe from 'stripe'
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!)

export async function POST(req: Request) {
  const sig = req.headers.get('stripe-signature')!
  let event: Stripe.Event
  try {
    event = stripe.webhooks.constructEvent(
      await req.text(),
      sig,
      process.env.STRIPE_WEBHOOK_SECRET!
    )
  } catch {
    return new Response('Invalid signature', { status: 400 })
  }
  // process event...
}
```

---

## A09 — Security Logging and Monitoring Failures

**Log security events, never sensitive data:**
```typescript
// GOOD: log the security event, not the secret value
logger.warn('auth.failed_login', {
  userId: user?.id ?? null,
  ip: req.ip,
  userAgent: req.headers['user-agent'],
  timestamp: new Date().toISOString(),
})

// BAD: leaks credentials into log pipeline
logger.info('login attempt', { email, password })
logger.error('stripe error', { error, apiKey: process.env.STRIPE_KEY })
```

**Alert thresholds to monitor:**
- 5+ failed logins for same account within 60 seconds → lock + alert
- Unusual geographic login (impossible travel)
- Admin actions (role changes, mass deletes) → audit log with actor
- 4xx spike > 10x baseline → potential scanning/fuzzing

---

## A10 — Server-Side Request Forgery (SSRF)

**Detection pattern:**
```
fetch\(req\.(body|query|params)
axios\.(get|post)\(req\.(body|query|params)
url\.parse\(req\.(body|query|params)
```

**Fix — validate and restrict outbound URLs:**
```typescript
import { URL } from 'url'

function isSafeUrl(raw: string): boolean {
  let parsed: URL
  try { parsed = new URL(raw) } catch { return false }

  // Block internal network ranges
  const blocked = [/^localhost$/i, /^127\./, /^10\./, /^172\.(1[6-9]|2\d|3[01])\./, /^192\.168\./]
  if (blocked.some(re => re.test(parsed.hostname))) return false

  // Only allow expected schemes
  return ['https:'].includes(parsed.protocol)
}

export async function fetchUserAvatar(avatarUrl: string) {
  if (!isSafeUrl(avatarUrl)) throw new Error('URL not allowed')
  return fetch(avatarUrl)
}
```

**File upload validation (preventing upload-based SSRF and injection):**
```typescript
const ALLOWED_MIME = new Set(['image/jpeg', 'image/png', 'image/webp'])
const MAX_BYTES = 5 * 1024 * 1024  // 5 MB

function validateUpload(file: File): void {
  if (file.size > MAX_BYTES) throw new Error('File exceeds 5 MB limit')
  if (!ALLOWED_MIME.has(file.type)) throw new Error('Unsupported file type')

  // Verify magic bytes match declared MIME type (don't trust file.type alone)
  // Use a library like file-type for server-side mime sniffing
}
```

---

## Rate Limiting

```typescript
import { Ratelimit } from '@upstash/ratelimit'
import { Redis } from '@upstash/redis'

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '60s'),
  analytics: true,
})

export async function POST(req: Request) {
  const identifier = req.headers.get('x-forwarded-for') ?? 'anonymous'
  const { success, limit, remaining, reset } = await ratelimit.limit(identifier)

  if (!success) {
    return Response.json({ error: 'Too many requests' }, {
      status: 429,
      headers: {
        'Retry-After': String(Math.ceil((reset - Date.now()) / 1000)),
        'X-RateLimit-Limit': String(limit),
        'X-RateLimit-Remaining': String(remaining),
      },
    })
  }
  // proceed
}
```
