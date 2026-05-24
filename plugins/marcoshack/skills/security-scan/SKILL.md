---
name: security-scan
description: Scan the codebase for leaked secrets, API security vulnerabilities, input validation issues, authentication bypass risks, and sensitive data exposure. Use when user asks to check for secrets, scan for security issues before commit, audit sensitive data, or review API security.
disable-model-invocation: false
allowed-tools: Grep, Glob, Read, Bash
---

# Security Scan

Perform a comprehensive security audit covering secret detection, API security vulnerabilities, and application security best practices aligned with OWASP API Security Top 10 (2023) and CWE standards.

---

## Part 1: Secret Detection

### 1.1 Identify Files to Scan

Use git to determine which files are tracked and would be committed:
```bash
# List all tracked files
git ls-files

# List untracked files not in .gitignore
git ls-files --others --exclude-standard

# Check current git status
git status --porcelain
```

### 1.2 Read .gitignore Configuration

Read `.gitignore` to verify sensitive files are properly excluded:
- `.env` files containing actual secrets
- Credential files (`.pem`, `.key`, `.p12`, `.pfx`, `.jks`)
- Private keys
- Build artifacts that might contain embedded secrets
- Database dumps or exports

### 1.3 Scan for Common Secret Patterns

Search for hardcoded secrets using pattern matching:

**API Keys and Tokens:**
- `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `DISCORD_WEBHOOK`, `GITHUB_TOKEN`
- `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`
- `GOOGLE_API_KEY`, `STRIPE_SECRET_KEY`, `SENDGRID_API_KEY`
- Generic patterns: `api[_-]?key.*=.*[a-zA-Z0-9]{20,}`
- Bearer tokens: `Bearer [a-zA-Z0-9_\-\.]+`
- Base64-encoded secrets: long base64 strings assigned to secret/key/token variables

**Credentials:**
- Database connection strings with embedded passwords
- OpenSearch/Elasticsearch credentials
- OAuth client secrets
- JWT signing secrets / HMAC keys
- Basic auth credentials

**Sensitive Data:**
- Private SSH keys (`-----BEGIN (RSA|EC|OPENSSH) PRIVATE KEY-----`)
- TLS/SSL private keys and certificates
- URLs with embedded credentials (`https://user:pass@host`)
- PGP private keys

### 1.4 Verify Environment Variable Usage

Check that all sensitive configuration uses environment variables:
- Go: `os.Getenv()`, `viper`, config structs with `env:` tags
- JavaScript/TypeScript: `process.env`
- Docker Compose: `${VAR}` substitution, not hardcoded values
- No hardcoded values for secrets in any language

### 1.5 Check Staged Changes

If files are staged for commit:
```bash
git diff --cached
```
Ensure no secrets are about to be committed.

### 1.6 Pattern-Based Secret Detection

Use grep to find potential secrets in tracked files:
```bash
git ls-files | xargs grep -iE "(api[_-]?key|password|secret|token|webhook).*[=:].*['\"]?[a-zA-Z0-9]{20,}"
```

---

## Part 2: API Security (OWASP API Security Top 10)

### 2.1 Broken Object-Level Authorization (API1:2023)

Scan for endpoints that access resources by ID without verifying the requester owns or has access to the resource.

**Check for:**
- Route handlers that extract IDs from URL params/body and pass directly to DB queries without authorization checks
- Missing ownership verification: `userID == resource.OwnerID` or role-based access checks
- Direct object references without ACL enforcement (IDOR vulnerabilities)
- Inconsistent authorization — some endpoints check, others don't for the same resource type

**Patterns to grep:**
- Handler functions that call repository methods directly (bypassing service-layer auth)
- Path parameters like `{id}`, `{userId}`, `{itemId}` used without authorization middleware or in-handler checks

### 2.2 Broken Authentication (API2:2023)

Scan for authentication weaknesses.

**Check for:**
- Endpoints missing authentication middleware — compare route registration against middleware chains
- Weak JWT configuration: missing expiration (`exp`), weak signing algorithms (`alg: none`, `HS256` with short keys), missing audience/issuer validation
- Tokens stored in localStorage without XSS protections (prefer httpOnly cookies)
- Missing token revocation / logout doesn't invalidate server-side
- Password handling: plaintext storage, weak hashing (MD5/SHA1 instead of bcrypt/argon2/scrypt), missing salt
- Missing brute-force protection on login endpoints (no rate limiting, no account lockout)
- Session fixation: tokens not rotated after login/privilege change

### 2.3 Broken Object Property-Level Authorization (API3:2023)

Scan for mass assignment and excessive data exposure.

**Check for:**
- **Mass assignment**: Request body decoded directly into DB model structs without allowlisting fields
  - Go: `json.Decode` into a model with sensitive fields (role, is_admin, created_at)
  - JS: `Object.assign(model, req.body)` or spread `{...req.body}` into DB updates
- **Excessive data exposure**: API responses returning full DB models including sensitive fields
  - Responses containing `password_hash`, `secret`, `internal_id`, `storage_key`, `deleted_at`
  - Missing DTOs / response structs — returning repository models directly to clients
  - `json:"-"` tags missing on sensitive Go struct fields

### 2.4 Unrestricted Resource Consumption (API4:2023)

Scan for missing rate limiting and resource abuse vectors.

**Check for:**
- No rate limiting middleware on authentication endpoints (login, register, password reset)
- No rate limiting on expensive operations (search, file upload, export, bulk operations)
- Missing pagination limits — unbounded `LIMIT` or no maximum page size
- File upload without size limits or type validation
- Missing request body size limits
- Regex patterns vulnerable to ReDoS (catastrophic backtracking)
- Unbounded query complexity (e.g., deeply nested GraphQL, unlimited JOINs)

### 2.5 Broken Function-Level Authorization (API5:2023)

Scan for privilege escalation and missing role checks.

**Check for:**
- Admin endpoints accessible without admin role verification
- Inconsistent role checks: e.g., `DELETE` requires admin but `PUT` on same resource doesn't
- Missing middleware for admin routes vs. applying checks only in handler logic
- Horizontal privilege escalation: user can perform actions on other users' resources
- Role checks that rely on client-provided role claims without server-side verification

### 2.6 Unrestricted Access to Sensitive Business Flows (API6:2023)

Scan for abuse of business logic.

**Check for:**
- No rate limiting on business-critical flows (account creation, invitations, exports)
- Missing CAPTCHA or bot detection on public-facing forms
- Enumeration vectors: different error messages for "user not found" vs "wrong password"
- Lack of idempotency keys on payment/mutation endpoints

### 2.7 Server-Side Request Forgery (API7:2023)

Scan for SSRF vulnerabilities.

**Check for:**
- User-supplied URLs passed to `http.Get()`, `fetch()`, `axios()`, `net/http` without validation
- URL parameters used in server-side requests without allowlisting domains
- Missing SSRF protections: no blocklist for internal IPs (`127.0.0.1`, `169.254.169.254`, `10.*`, `172.16-31.*`, `192.168.*`, `::1`, `fd00::/8`)
- Redirect following on user-supplied URLs without validation
- Image/file fetching from user-provided URLs

### 2.8 Security Misconfiguration (API8:2023)

Scan for insecure default configurations.

**Check for:**
- CORS: `Access-Control-Allow-Origin: *` in production, overly permissive allowed methods/headers
- Missing security headers: `X-Content-Type-Options`, `X-Frame-Options`, `Strict-Transport-Security`, `Content-Security-Policy`
- Debug mode / verbose errors enabled in production (stack traces, SQL errors returned to client)
- Default credentials left in config files or Docker Compose
- TLS/SSL: disabled certificate verification (`InsecureSkipVerify: true`, `NODE_TLS_REJECT_UNAUTHORIZED=0`)
- Unnecessary HTTP methods enabled (TRACE, OPTIONS returning too much info)
- Directory listing enabled on static file servers
- Permissive file permissions on sensitive config files

### 2.9 Improper Inventory Management (API9:2023)

Scan for shadow/undocumented endpoints and API versioning issues.

**Check for:**
- Debug/test endpoints left in production code (`/debug/pprof`, `/test/`, `/_internal/`)
- Deprecated API versions still accessible without deprecation headers
- Undocumented endpoints (routes registered but not in API docs/OpenAPI spec)
- Different authentication requirements between API versions

### 2.10 Unsafe Consumption of APIs (API10:2023)

Scan for insecure handling of third-party API responses.

**Check for:**
- Third-party API responses used without validation or sanitization
- Missing error handling on external API calls
- Trusting external data without type checking or schema validation
- Credentials for external APIs hardcoded or logged

---

## Part 3: Injection Vulnerabilities (CWE-89, CWE-78, CWE-79)

### 3.1 SQL Injection

**Check for:**
- String concatenation or `fmt.Sprintf` in SQL queries with user input
- Missing parameterized queries / prepared statements
- Dynamic `ORDER BY`, `LIMIT`, or `WHERE` clauses built from user input without allowlisting
- ORM raw query methods with unescaped input

**Patterns to grep:**
- `fmt.Sprintf("SELECT` or `fmt.Sprintf("INSERT` or `fmt.Sprintf("UPDATE` or `fmt.Sprintf("DELETE`
- `"SELECT * FROM " +` or `"WHERE " + userInput`
- `.Raw(` or `.Exec(` with string concatenation

### 3.2 Command Injection

**Check for:**
- `exec.Command()` with user-supplied arguments without sanitization
- Shell execution with string interpolation: `os/exec`, `child_process.exec()`
- Unsanitized input passed to system commands

### 3.3 Cross-Site Scripting (XSS)

**Check for:**
- `dangerouslySetInnerHTML` usage without sanitization (React)
- User input rendered without escaping in templates
- Reflected input in error messages or redirect URLs
- Missing `Content-Type` headers on API responses (should be `application/json`)

### 3.4 Path Traversal (CWE-22)

**Check for:**
- User input used in file paths without sanitization: `../` sequences
- Missing `filepath.Clean()` or equivalent on user-supplied paths
- File serving endpoints without root directory containment

---

## Part 4: Sensitive Data in Logs and Errors

### 4.1 Logging Sensitive Data

**Check for:**
- Passwords, tokens, API keys, or session IDs in log statements
- Request/response body logging that may include credentials or PII
- Full error details (stack traces, SQL queries, internal paths) logged at INFO level or returned to clients
- User PII (email, phone, SSN) logged without redaction

### 4.2 Error Response Leakage

**Check for:**
- Stack traces or internal error messages returned in API responses
- Database error details (table names, column names, constraint names) exposed to clients
- File system paths leaked in error messages
- Different HTTP status codes revealing resource existence (404 vs 403 — should be consistent)
- Version/technology disclosure in error responses or headers (`X-Powered-By`, `Server`)

---

## Part 5: Cryptographic Issues (CWE-327, CWE-328)

**Check for:**
- Weak hashing: MD5 or SHA1 used for passwords or security tokens
- Hardcoded encryption keys, IVs, or salts
- Insecure random number generation (`math/rand` instead of `crypto/rand` in Go, `Math.random()` in JS for security purposes)
- Missing HTTPS enforcement for sensitive endpoints
- Weak JWT algorithms (`none`, `HS256` with short secret)

---

## Scan Procedure

1. **Run Part 1** (Secret Detection) — always
2. **Run Parts 2-5** (API Security) — scan all handler, middleware, service, and route files
3. **Cross-reference** findings: e.g., an endpoint missing auth (2.2) that also has SQL injection (3.1) is critical
4. **Prioritize** by severity and exploitability

---

## Report Format

Structure the final report as:

```markdown
## Security Scan Complete - [SAFE/UNSAFE]

### Scan Scope
- Files scanned: N
- Categories: Secrets, API Security, Injection, Data Exposure, Crypto

---

### Critical Findings
(Must fix before commit/deploy)

| # | Category | File | Finding | Severity | CWE/OWASP |
|---|----------|------|---------|----------|------------|
| 1 | Auth Bypass | [handler/foo.go:42](handler/foo.go#L42) | Endpoint missing auth middleware | Critical | API2:2023 |

### High Findings
(Should fix soon)

| # | Category | File | Finding | Severity | CWE/OWASP |
|---|----------|------|---------|----------|------------|
| 1 | ... | ... | ... | High | ... |

### Medium / Low Findings
(Address in next iteration)

| # | Category | File | Finding | Severity | CWE/OWASP |
|---|----------|------|---------|----------|------------|
| 1 | ... | ... | ... | Medium | ... |

---

### Secrets Audit
- .env ignored: YES/NO
- Hardcoded secrets found: YES/NO
- Environment variables used correctly: YES/NO

### API Security Summary
- [ ] Object-level authorization (API1)
- [ ] Authentication (API2)
- [ ] Property-level authorization (API3)
- [ ] Resource consumption limits (API4)
- [ ] Function-level authorization (API5)
- [ ] Business flow abuse (API6)
- [ ] SSRF protections (API7)
- [ ] Security configuration (API8)
- [ ] API inventory (API9)
- [ ] Third-party API safety (API10)

---

**Status**: [SAFE/UNSAFE] — N critical, N high, N medium, N low findings
```

## Severity Classification

| Severity | Criteria |
|----------|----------|
| **Critical** | Direct exploitation possible: auth bypass, SQL injection, hardcoded secrets, RCE |
| **High** | Significant risk with some conditions: IDOR, mass assignment, SSRF, weak crypto for auth |
| **Medium** | Moderate risk or defense-in-depth gap: missing rate limits, verbose errors, missing headers |
| **Low** | Best practice violation, minimal direct risk: info disclosure in headers, missing CSP |

## Example Scans

**Full security scan:**
```
/security-scan
```

**Scan specific directories:**
```
/security-scan src/
/security-scan api/internal/handler/
```

**Before committing (staged files only):**
```
/security-scan --staged
```

**API-focused scan only:**
```
/security-scan --api
```

## Notes

- Focus on files tracked by git (or about to be committed)
- Read files using Read tool for accurate line numbers
- Use Grep for pattern matching across multiple files
- Provide clickable file references: `[file.py:42](file.py#L42)`
- Be concise but thorough
- Always err on the side of caution — flag suspicious patterns
- Reference OWASP/CWE identifiers for each finding so teams can look up remediation
- For each finding, include a concrete remediation suggestion
