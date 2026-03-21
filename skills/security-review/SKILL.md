---
name: security-review
description: This skill should be used when the user is implementing authentication, authorization, input handling, secrets management, API endpoints, file uploads, payment flows, or dependency management; or when reviewing code for OWASP Top 10 vulnerabilities, hardcoded secrets, injection risks, IDOR, or insecure cryptography.
version: 1.0.0
---

# Security Review

Expert-level security patterns covering OWASP Top 10 (2021), ASVS Level 1-2 controls, and concrete detection patterns for the most common vulnerability classes.

## When to Activate

- Adding or modifying authentication or session management
- Implementing authorization checks or RBAC/ABAC
- Handling any user-supplied input (forms, query params, file uploads, webhooks)
- Creating or modifying API endpoints
- Storing, transmitting, or logging sensitive data
- Managing secrets, API keys, or credentials
- Integrating third-party libraries or updating dependencies
- Performing a pre-deployment or pre-merge security review

---

## Five-Pass Review Process

Perform reviews in this order — earlier passes block later ones:

1. **Secrets pass** — scan for hardcoded credentials, keys, and tokens before anything else
2. **Input pass** — trace every user-controlled value from entry point to sink (DB, shell, HTML output)
3. **Auth pass** — verify every endpoint has appropriate authentication and authorization middleware
4. **Transport pass** — confirm HTTPS enforcement, security headers, and cookie flags
5. **Dependency pass** — run `npm audit` and check for outdated packages with known CVEs

---

## OWASP Top 10 (2021): Summary Table

| # | Name | One-Line Detection Signal |
|---|---|---|
| A01 | Broken Access Control | Route handlers missing auth middleware; DB queries not scoped to `req.user.id` |
| A02 | Cryptographic Failures | Passwords hashed with MD5/SHA1; secrets in source code; `http://` non-local URLs |
| A03 | Injection | Template literals in SQL queries; `execSync` with user input; `req.body` spread into DB query |
| A04 | Insecure Design | No rate limiting on mutating endpoints; business logic limits absent on server |
| A05 | Security Misconfiguration | `cors()` with no options (wildcard); `DEBUG=true` in prod; missing `helmet()` |
| A06 | Vulnerable Components | `npm audit` high/critical findings; unpinned CI action tags; no lock file in repo |
| A07 | Auth Failures | JWT stored in `localStorage`; missing `algorithms` array in `jwt.verify()`; no account lockout |
| A08 | Data Integrity Failures | Webhook payloads processed without signature verification; CI actions pinned to tags not SHAs |
| A09 | Logging Failures | Passwords or tokens in log statements; no logging of failed auth events |
| A10 | SSRF | `fetch(req.body.url)` or `axios.get(req.query.url)` without allowlist validation |

See [`references/owasp-checklist.md`](references/owasp-checklist.md) for full detection regex, TypeScript fix examples, Helmet.js/CORS configs, rate limiting, and SSRF prevention per item.

---

## Secrets Detection: Regex Patterns

Run these patterns in pre-commit hooks and CI to catch secrets before they reach the repository:

```
# Generic high-entropy secrets
(api[_-]?key|secret|password|token|private[_-]?key)\s*[=:]\s*['"][A-Za-z0-9+/=_\-]{8,}['"]

# Provider-specific formats
sk-[a-zA-Z0-9]{32,}          # OpenAI-style keys
ghp_[a-zA-Z0-9]{36}          # GitHub PATs
AKIA[0-9A-Z]{16}             # AWS access key IDs
-----BEGIN (RSA |EC )?PRIVATE KEY-----

# Weak hashing for passwords
createHash\(['"]md5['"]|createHash\(['"]sha1['"]

# HTTP (non-local) in source
http://(?!localhost|127\.0\.0\.1)
```

Run automated secret scanning:

```bash
# secretlint — file-level scan
npx @secretlint/secretlint "**/*"

# trufflehog — full git history scan
trufflehog git file://. --only-verified
```

---

## Input Validation: Schema-First Approach

Validate all user-supplied input at the API boundary using a schema library. Never trust `req.body` types.

```typescript
import { z } from 'zod'

const CreateOrderSchema = z.object({
  items: z.array(
    z.object({
      productId: z.string().uuid(),
      quantity: z.number().int().min(1).max(100),
    })
  ).min(1).max(50),
  couponCode: z.string().regex(/^[A-Z0-9]{4,16}$/).optional(),
  currency: z.enum(['USD', 'EUR', 'GBP']),
})

export async function POST(req: Request) {
  const result = CreateOrderSchema.safeParse(await req.json())
  if (!result.success) {
    // Return sanitized error — do not echo raw user input back
    return Response.json({ error: 'Invalid request', details: result.error.flatten() }, { status: 400 })
  }
  return processOrder(result.data)
}
```

---

## Secrets Management

```
Vault (HashiCorp) / AWS Secrets Manager / GCP Secret Manager
           |
     injected at runtime
           |
      process.env.*          <- only access point in code
           |
  verified at startup (fail fast if missing)
```

Validate required environment variables at startup:

```typescript
const REQUIRED_ENV = [
  'DATABASE_URL', 'JWT_SECRET', 'STRIPE_SECRET_KEY', 'REDIS_URL'
] as const

for (const key of REQUIRED_ENV) {
  if (!process.env[key]) {
    throw new Error(`Missing required environment variable: ${key}`)
  }
}
```

---

## Pre-Deployment Security Checklist

Run before every production deployment:

**Secrets**
- [ ] No hardcoded secrets (`secretlint` scan passes)
- [ ] All secrets loaded from environment variables
- [ ] `.env*` files in `.gitignore`; no secrets in git history

**Input Handling**
- [ ] All API inputs validated with a schema (Zod, Joi, Yup)
- [ ] File uploads restricted by size and MIME type (with magic byte check)
- [ ] No string concatenation in SQL or OS commands

**Authentication / Authorization**
- [ ] Passwords hashed with bcrypt (>=12 rounds) or argon2id
- [ ] JWTs: short expiry, explicit algorithm, stored in httpOnly cookie
- [ ] Every endpoint has appropriate auth middleware
- [ ] IDOR: all DB queries scoped to `req.user.id`

**Transport and Headers**
- [ ] HTTPS enforced; HSTS header set
- [ ] CORS origin allowlist configured (no wildcard)
- [ ] Security headers: CSP, X-Frame-Options, X-Content-Type-Options
- [ ] Cookies: HttpOnly, Secure, SameSite=Strict

**Error Handling and Logging**
- [ ] No stack traces or internal errors exposed to clients
- [ ] No passwords, tokens, or PII in log output
- [ ] Security events (login failures, auth errors) logged with IP and timestamp

**Dependencies**
- [ ] `npm audit` exits with 0 high/critical findings
- [ ] Lock file committed; CI uses `npm ci`
- [ ] Outdated packages reviewed and updated

**Rate Limiting and Abuse Prevention**
- [ ] Rate limiting on auth endpoints (login, register, password reset)
- [ ] Rate limiting on expensive endpoints (search, exports, AI calls)
- [ ] Account lockout after repeated failed logins

---

## References

- [`references/owasp-checklist.md`](references/owasp-checklist.md) — Full OWASP A01-A10 with detailed detection regex, TypeScript fix examples, Helmet.js/CORS configs, rate limiting, SSRF prevention
- [`references/auth-patterns.md`](references/auth-patterns.md) — bcrypt/argon2id hashing, JWT sign/verify with algorithm array, refresh token rotation, RBAC middleware, timing-safe comparison, session management
