---
name: security-review
description: Use before merging any code that handles user input, authentication, authorization, data storage, or external APIs
---

# Security Review

## Overview

Security issues found in code review cost 10x less to fix than issues found in production. This skill is a systematic checklist to catch common vulnerabilities before they ship.

**Core principle:** Treat all external input as malicious until proven safe. Defense in depth.

**Announce at start:** "I'm using the security-review skill."

## When to Use

Run this on any change that touches:
- User input handling (forms, APIs, file uploads)
- Authentication or session management
- Authorization / access control
- Database queries
- File system operations
- External API calls or webhooks
- Cryptography or secrets
- Dependency updates

**Run it even if the change looks small.** Security bugs often live in the interaction between changes.

## The Checklist

Work through each section. Mark every item explicitly: ✅ safe, ⚠️ needs attention, or N/A.

### 1. Input Validation

- [ ] All user-supplied input is validated before use
- [ ] Validation happens server-side, not just client-side
- [ ] Input length, type, and format are enforced
- [ ] Unexpected input fails closed (reject, not accept)
- [ ] File uploads: type checked by content, not extension; stored outside webroot

**Red flags:**
```python
# DANGEROUS — user controls the query
query = f"SELECT * FROM users WHERE name = '{user_input}'"

# SAFE — parameterized query
query = "SELECT * FROM users WHERE name = ?"
cursor.execute(query, (user_input,))
```

### 2. Injection

- [ ] SQL: all queries use parameterized statements or ORM
- [ ] HTML: all user content is escaped before rendering (XSS)
- [ ] Shell: no user input passed to `exec()`, `system()`, `subprocess` without strict allowlist
- [ ] Path traversal: file paths are normalized and restricted to allowed directories
- [ ] Template injection: user content not interpolated into template strings

```javascript
// XSS — DANGEROUS
element.innerHTML = userInput;

// XSS — SAFE
element.textContent = userInput;
```

### 3. Authentication

- [ ] Passwords are hashed with bcrypt, scrypt, or Argon2 (not MD5, SHA-1, plain SHA-256)
- [ ] Password reset tokens are cryptographically random and expire
- [ ] Failed login attempts are rate-limited
- [ ] Session tokens are sufficiently random (128+ bits entropy)
- [ ] Sessions are invalidated on logout
- [ ] "Remember me" tokens are stored hashed, not plaintext

### 4. Authorization

- [ ] Every endpoint checks authorization, not just authentication
- [ ] Authorization is checked server-side on every request
- [ ] Indirect object references are validated (user can only access their own resources)
- [ ] Admin/privileged routes are protected separately
- [ ] Authorization logic is centralized, not scattered

**Common mistake — IDOR:**
```python
# DANGEROUS — user can change the ID to access others' data
def get_order(order_id):
    return db.query("SELECT * FROM orders WHERE id = ?", order_id)

# SAFE — scoped to current user
def get_order(order_id, current_user):
    return db.query(
        "SELECT * FROM orders WHERE id = ? AND user_id = ?",
        order_id, current_user.id
    )
```

### 5. Secrets and Configuration

- [ ] No secrets, API keys, or credentials in source code or committed files
- [ ] Secrets are loaded from environment variables or a secrets manager
- [ ] `.env` files are in `.gitignore`
- [ ] Error messages don't expose stack traces, internal paths, or system info to users
- [ ] Debug mode / verbose logging disabled in production config

```bash
# Check for accidentally committed secrets
git log --all --full-history -- "*.env"
git grep -i "password\s*=" -- "*.py" "*.js" "*.ts"
```

### 6. Cryptography

- [ ] Using well-established libraries (not rolling your own crypto)
- [ ] TLS used for all external communication
- [ ] Encryption keys are rotatable
- [ ] Random numbers use `secrets` / `crypto.randomBytes` (not `Math.random()` or `random.random()`)
- [ ] JWTs: `alg` is explicitly specified and validated (not trusted from token header)

### 7. Dependencies

- [ ] New dependencies checked for known CVEs (`npm audit`, `pip-audit`, `snyk`)
- [ ] Dependencies pinned to specific versions in lockfile
- [ ] Dependency sources are trusted (official registries, not arbitrary git URLs)

```bash
npm audit
pip-audit
```

### 8. Logging and Monitoring

- [ ] Security-relevant events are logged: login success/failure, permission denied, admin actions
- [ ] Logs do NOT contain passwords, tokens, or PII
- [ ] Log injection prevented (user input sanitized before logging)

### 9. Rate Limiting and DoS

- [ ] Expensive operations (search, export, file processing) are rate-limited
- [ ] File upload size is capped
- [ ] Pagination is required on list endpoints (no unbounded queries)
- [ ] Timeouts set on external calls

### 10. CSRF and CORS

- [ ] State-changing requests (POST/PUT/DELETE) are protected against CSRF
- [ ] CORS policy is explicit and restrictive (not `Access-Control-Allow-Origin: *` on authenticated endpoints)
- [ ] `SameSite` cookie attribute set appropriately

## Severity Classification

| Severity | Examples | Action |
|----------|---------|--------|
| **Critical** | SQLi, RCE, auth bypass, plaintext passwords | Block merge. Fix now. |
| **High** | XSS, IDOR, missing auth on sensitive endpoint | Block merge. Fix before ship. |
| **Medium** | Missing rate limit, verbose errors, weak session config | Fix in current sprint. |
| **Low** | Missing security headers, dependency with low-severity CVE | Track and fix soon. |

## Output Format

After running the checklist, produce a summary:

```
## Security Review Summary

**Reviewed:** [what was changed]
**Risk level:** Critical / High / Medium / Low / Clean

### Findings

| # | Severity | Issue | File:Line | Recommendation |
|---|----------|-------|-----------|---------------|
| 1 | High     | IDOR on /api/orders | orders.py:45 | Scope query to current user |
| 2 | Medium   | No rate limit on /api/login | auth.py:12 | Add rate limiting middleware |

### Confirmed Safe
- Input validation: ✅
- SQL queries: ✅ (all parameterized)
- Secrets management: ✅
...

### Not Applicable
- File uploads: N/A (no file handling in this change)
```

## Related Skills

- **superpowers:requesting-code-review** — general code review (security review is a specialization)
- **superpowers:verification-before-completion** — verify fixes work before marking resolved
- **superpowers:systematic-debugging** — if a security issue is actively being exploited
