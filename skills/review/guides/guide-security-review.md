# Security Review Guide

Security-focused code review guide covering OWASP Top 10:2025, secure coding practices, and defense-in-depth strategies.

## Table of Contents

- [OWASP Top 10](#owasp-top-10)
- [Input Validation](#input-validation)
- [Authentication & Authorization](#authentication--authorization)
- [Data Protection](#data-protection)
- [Dependency Security](#dependency-security)
- [Logging & Monitoring](#logging--monitoring)
- [Security Review Checklist](#security-review-checklist)

---

## OWASP Top 10:2025

> The OWASP Top 10:2025 introduces updates based on latest threat data. Key changes include emphasis on supply chain security and application resilience.

### 1. Injection

**What it is:** Attacker sends untrusted data to an interpreter as part of a command or query.

**Common patterns:**
```python
# ❌ SQL Injection vulnerability
cursor.execute(f"SELECT * FROM users WHERE name = '{username}'")

# ✅ Parameterized query
cursor.execute("SELECT * FROM users WHERE name = %s", (username,))
```

```javascript
// ❌ Command injection
exec(`git checkout ${branchName}`);

// ✅ Use array form or whitelist
exec(['git', 'checkout', branchName]);
```

**Review checklist:**
- [ ] All user input parameterized in SQL queries
- [ ] Shell commands use array form, not string interpolation
- [ ] Template engines auto-escape output
- [ ] LDAP/NoSQL injection prevention in place

### 2. Broken Authentication

**What it is:** Attackers compromise authentication methods to impersonate users.

**Review checklist:**
- [ ] Passwords hashed with bcrypt/scrypt/argon2 (not MD5/SHA1)
- [ ] Rate limiting on login endpoints
- [ ] Account lockout after failed attempts
- [ ] Secure session management (HttpOnly, Secure cookies)
- [ ] Multi-factor authentication for sensitive operations

### 3. Sensitive Data Exposure

**What it is:** Failure to protect sensitive data at rest or in transit.

**Review checklist:**
- [ ] HTTPS/TLS enforced everywhere
- [ ] No sensitive data in URLs or logs
- [ ] Database encryption for PII
- [ ] API keys not hardcoded (use environment variables)
- [ ] Cache headers set appropriately

### 4. XML External Entities (XXE)

**What it is:** Attacker injects external entity references in XML input.

```xml
<!-- ❌ Dangerous XML parser configuration -->
<?xml version="1.0"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>

<!-- ✅ Disable DTD processing -->
parser.setFeature('http://apache.org/xml/features/disallow-doctype-decl', True)
```

### 5. Broken Access Control

**What it is:** Users can access resources or perform actions they shouldn't.

**Review checklist:**
- [ ] Authorization checks on every endpoint
- [ ] No IDOR (Insecure Direct Object Reference) vulnerabilities
- [ ] Role-based access control properly implemented
- [ ] Admin endpoints protected
- [ ] API rate limiting per user/role
- [ ] Horizontal privilege escalation prevented

### 6. Vulnerable and Outdated Components (formerly A06)

**What it is:** Using components with known vulnerabilities.

**Review checklist:**
- [ ] Dependencies scanned for known CVEs (npm audit, pip-audit, cargo audit)
- [ ] Lock files used and committed
- [ ] Automated dependency scanning in CI/CD
- [ ] Transitive dependencies audited
- [ ] Supply chain integrity verified (signed packages)

---

## Input Validation

### Principles

1. **Validate on input, validate on output** - Never trust client-side validation alone
2. **Whitelist over blacklist** - Accept only known-good values
3. **Fail securely** - Deny by default

### Common Patterns

```python
# ❌ Invalidating user input directly in HTML
def render_profile(username: str) -> str:
    return f"<h1>{username}</h1>"  # XSS vulnerability!

# ✅ Sanitize/escape output
from markupsafe import escape
def render_profile(username: str) -> str:
    return f"<h1>{escape(username)}</h1>"
```

```typescript
// ❌ Using user input directly in regex (ReDoS vulnerability)
function validateInput(input: string) {
    const regex = new RegExp(`^${input}$`);  // Dangerous!
    return regex.test(data);
}

// ✅ Use well-tested validation libraries
import isEmail from 'validator/lib/isEmail';
function validateEmail(email: string): boolean {
    return isEmail(email);
}
```

### Validation by Type

| Input Type | Validation Method | Example |
|------------|------------------|---------|
| Email | RFC 5322 compliant regex or library | `isEmail()` |
| URL | Whitelist protocols (http, https) | Protocol check + domain validation |
| File Upload | MIME type check + extension whitelist + size limit | Content-type verification |
| Numbers | Range validation + type checking | `min`, `max`, `type` checks |
| Dates | ISO 8601 format + range validation | Format parsing + business logic checks |

---

## Authentication & Authorization

### Password Security

```python
# ❌ Weak password hashing
import hashlib
password_hash = hashlib.md5(password.encode()).hexdigest()

# ✅ Strong password hashing
from passlib.hash import bcrypt
password_hash = bcrypt.hash(password)
```

### JWT Best Practices

```javascript
// ❌ Insecure JWT configuration
const token = jwt.sign({ userId: user.id }, 'secret', { expiresIn: '365d' });

// ✅ Secure JWT configuration
const token = jwt.sign(
    { userId: user.id, iat: Math.floor(Date.now() / 1000) },
    process.env.JWT_SECRET,
    { expiresIn: '1h', algorithm: 'RS256' }
);
```

### Session Security

| Setting | Recommended Value | Purpose |
|---------|------------------|---------|
| HttpOnly | true | Prevents JavaScript access to cookies |
| Secure | true | Only sent over HTTPS |
| SameSite | Strict or Lax | CSRF protection |
| Max-Age | Short (1-24 hours) | Limits session lifetime |

---

## Data Protection

### Encryption at Rest

```python
# ❌ Storing plaintext passwords
user.password = "password123"

# ✅ Hash passwords
from passlib.hash import bcrypt
user.password = bcrypt.hash("password123")

# Verify during login
assert bcrypt.verify("password123", user.password)
```

### Encryption in Transit

- TLS 1.2+ required
- Certificate pinning for mobile apps
- HSTS headers set

### Data Minimization

- Only collect necessary personal data
- Anonymize/pseudonymize when possible
- Set data retention policies
- Implement right-to-deletion

---

## Dependency Security

### Common Vulnerabilities

| Type | Example | Prevention |
|------|---------|------------|
| Known CVEs | Log4Shell (CVE-2021-44228) | Regular dependency scans |
| Supply chain attacks | Compromised npm package | Lock files, verified signatures |
| Transitive dependencies | Vulnerable transitive dep | Audit all layers |

### Tools

| Tool | Language | Purpose |
|------|----------|---------|
| `npm audit` | JavaScript | Dependency vulnerability scan |
| `pip-audit` | Python | Dependency vulnerability scan |
| `cargo audit` | Rust | Dependency vulnerability scan |
| `snyk` | Multi | Comprehensive security scanning |
| `trivy` | Multi | Container and filesystem scanning |

---

## Logging & Monitoring

### What to Log

- Authentication attempts (success and failure)
- Authorization failures
- Input validation failures
- Sensitive operations (password changes, data exports)
- System errors (without sensitive data)

### What NOT to Log

- Passwords or password hashes
- Credit card numbers (full PAN)
- Social security numbers / national IDs
- API keys or tokens
- Full request/response bodies with PII

### Log Security

```python
# ❌ Logging sensitive data
logger.info(f"User login: {username}, password: {password}")

# ✅ Log without sensitive data
logger.info(f"User login: {username} from IP {ip_address}")
```

---

## Security Review Checklist

### 🔴 Critical (Must Fix)

- [ ] No SQL/command injection vulnerabilities
- [ ] Passwords hashed with bcrypt/scrypt/argon2
- [ ] HTTPS enforced everywhere
- [ ] No hardcoded secrets or API keys
- [ ] Authentication required for protected routes
- [ ] Authorization checks on all sensitive operations

### 🟡 Important (Should Fix)

- [ ] Rate limiting on authentication endpoints
- [ ] Input validation and output encoding
- [ ] Secure cookie/session configuration
- [ ] Dependencies up to date with no known CVEs
- [ ] Error messages don't leak internal details
- [ ] CORS properly configured

### 🟢 Nice to Have (Should Consider)

- [ ] Multi-factor authentication
- [ ] Content Security Policy headers
- [ ] Security headers (X-Frame-Options, X-Content-Type-Options, etc.)
- [ ] Automated dependency scanning in CI/CD
- [ ] Penetration testing scheduled
- [ ] Security documentation and incident response plan

---

## Reference Resources

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [CWE Database](https://cwe.mitre.org/)
- [SANS Secure Coding Standards](https://www.sans.org/secure-coding/)
- [Mozilla Security Observatory](https://observatory.mozilla.org/)
