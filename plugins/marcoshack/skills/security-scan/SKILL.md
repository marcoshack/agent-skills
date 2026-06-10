---
name: security-scan
description: Scan the codebase for leaked secrets, API security vulnerabilities, input validation issues, authentication bypass risks, sensitive data exposure, supply-chain and CI/CD weaknesses, container/infrastructure misconfiguration, and frontend token-handling issues. Use when user asks to check for secrets, scan for security issues before commit, audit sensitive data, review API security, or run a full security audit.
disable-model-invocation: false
allowed-tools: Grep, Glob, Read, Bash
---

# Security Scan

Perform a security audit aligned with: OWASP API Security Top 10 (2023), OWASP Top 10 (2021), OWASP ASVS 4.0, OWASP Secure Headers, CWE, NIST SSDF (SP 800-218), SLSA, and CIS Docker Benchmark.

---

## Scan Modes

**Full scan (default for `/security-scan`)** — run all applicable parts against the whole repository.

**Staged/diff scan (`--staged`, pre-commit, or when invoked by the commit skill)** — run Part 1 (secrets) against the diff, plus any other part whose subject files are touched by the diff (e.g. a change to a route file triggers Part 2 checks on that file; a Dockerfile change triggers Part 9 checks on it).

> **Diff-scan limitation — state this in the report.** A diff scan checks "is this change safe", not "is the system safe". It cannot detect *missing* controls (absent authorization, absent rate limiting, absent dependency scanning, missing security headers) unless the diff touches the relevant file. When running a diff scan, end the report with: the last time a full scan should be/was run, and a recommendation to run `/security-scan` (full) periodically — at minimum before releases and after auth/infra changes.

## Scope: Files to Scan

Do NOT limit the scan to API handler/service files. The full scan covers:

| Surface | Typical files |
|---------|---------------|
| API backend | handlers, middleware, services, repositories, route registration |
| Frontend SPA | API clients, auth contexts/hooks, token storage, HTML-injection sites |
| Background workers / queue consumers | worker mains, dispatchers, task handlers (any language) |
| Real-time | SSE/WebSocket hubs, subscription handlers |
| Database | migrations, schema files (for auth-hot-path indexes, see 6.4) |
| Containers | `Dockerfile*`, `docker-compose*.yml`, `.dockerignore` |
| CI/CD | `.github/workflows/*`, GitLab CI, pipeline definitions |
| Edge / proxy | nginx/Caddy/Traefik configs, CDN config |
| Host / deploy | systemd units, install/deploy scripts, Makefiles, Terraform/K8s manifests |

## Stack Detection and Applicability

Before scanning, detect the stack (languages, frameworks, infrastructure) from file extensions, manifests (`go.mod`, `package.json`, `pyproject.toml`, `Cargo.toml`, etc.), and config files. Then:

- Run only the parts/checks applicable to the detected stack. Examples: skip React-specific XSS checks if there is no frontend; skip CP-SAT/worker checks if there are no queue consumers; skip nginx checks if there is no proxy config.
- **List skipped sections in the report** under "Not applicable" with a one-line reason — so coverage gaps are visible, not silent.
- For polyglot repos (e.g. Go API + React frontend + Python worker), run each part against each applicable component.

## Detection Principles

These four principles matter more than any individual grep pattern:

1. **Hunt for missing controls, not just bad patterns.** Most real-world findings are absences: no authorization on a subscription, no `USER` in a Dockerfile, no Dependabot config, no token revocation, no panic recovery in a worker pool. For each surface, ask "what control SHOULD exist here?" and verify it does. Grep finds presence; only deliberate inventory finds absence.
2. **Trace flows end-to-end; don't grep-and-stop.** Read the full authentication flow (issue → validate → refresh → revoke), the full request path (proxy → middleware chain → handler → repository), and the full OAuth dance (authorize → state → callback). Logic flaws (state not bound to session, rate limiter keyed on a spoofable header, JWT skipping the is-user-still-active check that API keys perform) are only visible when you read the whole flow.
3. **Read configs semantically, not lexically.** A header being *present* in a config is not the same as it being *served* (e.g. nginx `add_header` is NOT inherited into `location` blocks that have their own `add_header`). Apply each tool's actual semantics.
4. **Verify before reporting.** For high-severity claims, confirm exploitability by reading the actual code path (and note "verified" in the finding). Flag uncertain findings as "needs verification" rather than dropping or overstating them.

---

## Part 1: Secret Detection

(See `checklist.md` in this skill directory for the full pattern checklist.)

### 1.1 Identify Files to Scan

```bash
git ls-files                              # tracked files
git ls-files --others --exclude-standard  # untracked, not ignored
git status --porcelain
```

### 1.2 Read .gitignore Configuration

Verify sensitive files are excluded: `.env` files, credential files (`.pem`, `.key`, `.p12`, `.pfx`, `.jks`), private keys, build artifacts, database dumps.

### 1.3 Scan for Common Secret Patterns

**API keys and tokens:** provider-named vars (`ANTHROPIC_API_KEY`, `AWS_SECRET_ACCESS_KEY`, `STRIPE_SECRET_KEY`, …), generic `api[_-]?key.*=.*[a-zA-Z0-9]{20,}`, `Bearer` tokens, long base64 strings assigned to secret/key/token variables.

**Credentials:** DB connection strings with embedded passwords, OAuth client secrets, JWT/HMAC signing secrets, basic auth.

**Cryptographic material:** private keys (`-----BEGIN .* PRIVATE KEY-----`), TLS certs, URLs with embedded credentials (`scheme://user:pass@host`).

### 1.4 Verify Environment Variable Usage

All sensitive config from env vars (Go `os.Getenv`/viper, JS `process.env`, Python `os.environ`); Docker Compose uses `${VAR}` substitution; secrets are mandatory with fail-fast validation, not insecure defaults.

### 1.5 Check Staged Changes

```bash
git diff --cached
```

### 1.6 Pattern-Based Sweep

```bash
git ls-files | xargs grep -iE "(api[_-]?key|password|secret|token|webhook).*[=:].*['\"]?[a-zA-Z0-9]{20,}"
```

### 1.7 Secrets Leaked Through Tooling

- Install/deploy scripts that `echo` generated passwords or tokens to stdout (lands in scrollback, CI logs, screen shares) — print "written to .env" instead
- Secrets in Makefile targets, CI workflow files (must use the secrets context), or docker build `ARG`s (persist in image history)

---

## Part 2: API Security (OWASP API Security Top 10)

### 2.1 Broken Object-Level Authorization (API1:2023)

**Check for:**
- Route handlers that extract IDs from URL params/body and pass directly to DB queries without ownership/tenancy checks (`userID == resource.OwnerID`, namespace membership, role checks)
- Inconsistent authorization — some endpoints check, others don't, for the same resource type
- **Real-time subscriptions (commonly missed):** SSE/WebSocket endpoints that authenticate the *connection* but not the *topic/channel*. If topics are predictable (`event:{id}`, `user:{id}`), any authenticated user can subscribe to other tenants' streams. Verify the hub/subscription handler consults the caller's auth info per topic, not just per connection.
- Public "share by link" endpoints: token entropy (≥128 bits), uniform 404 on bad tokens, and **PII minimization** — does the public payload expose more personal data (full names, avatars, emails) than the feature needs? Suggest data minimization and `Referrer-Policy: no-referrer` on share responses.

### 2.2 Broken Authentication (API2:2023)

**Check for:**
- Endpoints missing authentication middleware — compare route registration against middleware chains
- Weak JWT configuration: missing `exp`, `alg: none`, missing algorithm allow-list (algorithm-confusion guard), missing audience/issuer validation
- **Revocation parity (trace the flow):** stateless JWTs with no server-side revocation — logout is a no-op, disabled/demoted users keep access until expiry. Compare credential types: if API keys re-check `IsActive` per request but JWTs don't, flag the asymmetry. Look for `tokens_valid_after` / `jti` denylist; check token TTL (24h+ access tokens without refresh = finding); check whether a refresh endpoint exists but is unused by the client.
- **OAuth/OIDC flow (trace authorize → callback):**
  - `state` must be a per-session random nonce stored client-side (`Secure; HttpOnly; SameSite` cookie) and exact-matched at callback. An HMAC-of-timestamp state that any callback can replay within a window defeats CSRF protection → login CSRF / code injection.
  - PKCE for public clients; exact-match redirect URI validation; account-linking via verified-email paths reviewed for takeover
- **Rate-limiter keying / trusted proxies:** if the rate limiter (or any security decision) keys on `X-Forwarded-For` / `X-Real-IP`, verify those headers are only honored from configured trusted proxy IPs. Attacker-rotated headers = fresh bucket per request = full bypass of login/reset throttling.
- Password handling: plaintext, MD5/SHA1 instead of bcrypt/argon2/scrypt, missing brute-force protection on login
- Session fixation: tokens not rotated after login/privilege change

### 2.3 Broken Object Property-Level Authorization (API3:2023)

**Check for:**
- **Mass assignment:** request body decoded directly into DB models (Go `json.Decode` into structs with `role`/`is_admin`; JS `{...req.body}` into updates)
- **Excessive data exposure:** responses returning full DB models (`password_hash`, `internal_id`, `storage_key`); missing DTOs; missing `json:"-"` on sensitive Go fields

### 2.4 Unrestricted Resource Consumption (API4:2023)

**Check for:**
- Rate limiting **coverage map**: list all public/expensive endpoints (auth, share links, discovery/search, file upload, export, SSE) and verify each is covered. "Rate limiting exists on /auth/*" is not "rate limiting exists."
- Per-user/per-IP connection caps on SSE/WebSocket hubs (unbounded client maps = connection-exhaustion DoS)
- Missing pagination limits; file upload without size/type limits; missing request body size limits; ReDoS-prone regexes
- **Background workers must validate input bounds even from trusted internal callers** — e.g. a solver/processor whose memory grows combinatorially with payload size needs explicit maximums in its own validation, raising a non-retryable error
- Defense in depth: is there also a proxy-layer limit (see 9.3)?

### 2.5 Broken Function-Level Authorization (API5:2023)

**Check for:**
- Admin endpoints without admin role verification; inconsistent role checks across methods on the same resource
- Horizontal privilege escalation; role checks trusting client-supplied claims without server-side verification

### 2.6 Unrestricted Access to Sensitive Business Flows (API6:2023)

**Check for:**
- No rate limiting on account creation, invitations, exports; enumeration vectors (distinct "user not found" vs "wrong password"); missing idempotency keys on payment/mutation endpoints

### 2.7 Server-Side Request Forgery (API7:2023)

**Check for:**
- User-supplied URLs passed to `http.Get()`/`fetch()`/`axios()` without domain allow-listing
- Missing internal-IP blocklist (`127.0.0.1`, `169.254.169.254`, RFC1918, `::1`, `fd00::/8`); redirect following on user URLs; image/file fetching from user-provided URLs

### 2.8 Security Misconfiguration (API8:2023)

**Check for:**
- CORS `*` in production; missing security headers (verify they are actually *served*, see 9.3)
- Debug/verbose errors in production; default credentials in configs; `InsecureSkipVerify: true` / `NODE_TLS_REJECT_UNAUTHORIZED=0`
- Responses echoing stored/user-controlled `Content-Type` instead of a fixed type (e.g. avatar served with content-type from object-store metadata — hard-code it)
- **Network binding:** auxiliary servers (health, metrics, debug) binding `0.0.0.0` instead of the container/internal network; endpoints answering on any path/request line

### 2.9 Improper Inventory Management (API9:2023)

**Check for:**
- Debug/test endpoints in production (`/debug/pprof`, `/_internal/`); deprecated API versions without deprecation headers; undocumented routes; auth differences between versions

### 2.10 Unsafe Consumption of APIs (API10:2023)

**Check for:**
- Third-party API responses used without validation; missing error handling on external calls; external-API credentials hardcoded or logged

---

## Part 3: Injection Vulnerabilities

### 3.1 SQL Injection (CWE-89)

- String concatenation or `fmt.Sprintf` in queries with user input (note: `Sprintf` building only `$N` placeholder *indices* is safe — read before flagging)
- Dynamic `ORDER BY`/`WHERE` from user input without allow-listing; ORM raw-query methods with unescaped input
- Grep: `fmt.Sprintf("SELECT|INSERT|UPDATE|DELETE`, `"WHERE " +`, `.Raw(`/`.Exec(` with concatenation

### 3.2 Command Injection (CWE-78)

- `exec.Command()`/`child_process.exec()`/`subprocess` with user-supplied arguments; shell string interpolation

### 3.3 Cross-Site Scripting (CWE-79)

- `dangerouslySetInnerHTML` without sanitization (note mitigations like mermaid `securityLevel: 'strict'` but flag fragility)
- Markdown/anchor renderers passing arbitrary `href` without a scheme allow-list (`javascript:` URIs bypass react-markdown defaults when components are overridden); missing `rel="noopener noreferrer"`
- User input reflected in templates/errors/redirects; missing `Content-Type` on API responses

### 3.4 Path Traversal (CWE-22)

- User input in file paths without `filepath.Clean()`/containment; file-serving endpoints without root confinement

### 3.5 CRLF / Header Injection (CWE-93, CWE-113) — commonly missed

- **Email header injection:** code building MIME messages by string concatenation (`"Subject: " + subject + "\r\n"`) where the subject/recipient/from-name includes user- or admin-controlled values (event names, display names). Newlines inject `Bcc:` headers or body content. Verify the input validation layer rejects control characters, or the mail layer strips `\r\n`.
- HTTP response splitting: user input in response headers (redirects with user-controlled `Location`)
- Log injection: user input written to logs without newline escaping (forged log entries)

---

## Part 4: Sensitive Data Exposure (Logs, Errors, URLs)

### 4.1 Logging Sensitive Data

- Passwords/tokens/API keys/session IDs in log statements; request/response body logging with credentials or PII; full query strings logged when tokens travel in URLs

### 4.2 Error Response Leakage

- Stack traces / DB error details / file paths in API responses; 404 vs 403 revealing resource existence; `X-Powered-By`/`Server` disclosure

### 4.3 Tokens and Secrets in URLs (CWE-598)

- Bearer/session tokens in query strings (common with `EventSource`/SSE since it can't set headers, and with download/share links): they land in proxy/access logs, browser history, and Referer headers
- Fix patterns to suggest: cookie-based auth for the stream, or a short-lived single-purpose ticket token exchanged for the real session

---

## Part 5: Cryptographic Issues (CWE-327, CWE-328)

- MD5/SHA1 for passwords or security tokens; hardcoded keys/IVs/salts
- `math/rand` (Go) / `Math.random()` (JS) / `random` (Python) for security values instead of CSPRNG
- Weak JWT algorithms (`none`, short-secret `HS256`); missing HTTPS enforcement
- Positive checks (note in report when present): AEAD modes with random nonces, HKDF key separation, constant-time comparison for MACs/tokens, ≥128-bit random tokens

---

## Part 6: Real-Time, Workers, and Availability (DoS / CWE-248 / CWE-400)

Crash- and resource-safety of async surfaces is a security property: one poison message or one panic should never take down the service.

### 6.1 Real-Time Hubs (SSE / WebSocket)

- Per-topic authorization (see 2.1) and per-user connection caps (see 2.4)
- **Concurrency safety:** double-close of client channels between disconnect and shutdown paths (panic = process down); shared maps mutated without locks; sends on closed channels. If the hub package has zero tests, flag it — concurrency-sensitive code with no `-race` tests is a finding in itself.

### 6.2 Queue Consumers / Background Workers

- **Panic/exception recovery:** worker pools running tasks in goroutines/threads without `recover()`/try-except — one bad task crashes the whole worker (compare: does the HTTP path have recovery middleware while the worker path has none?)
- **Ack-deadline vs runtime:** ack timeout (`AckWait`, `ack_wait`, visibility timeout) shorter than worst-case task runtime → redelivery and duplicate execution; long tasks should heartbeat (in-progress acks)
- **Poison messages:** non-retryable errors (malformed payload, validation failures) NAK'd into retry loops instead of terminated + reported; no terminal-failure handling at max delivery attempts
- **Blocking the delivery thread:** consumer callbacks blocking on a saturated pool (stalls heartbeats / triggers slow-consumer disconnects)
- **Shutdown:** tasks that can't observe cancellation (context never cancelled, shutdown ctx discarded); CPU-bound work on an async event loop blocking health endpoints and signal handling (Python: missing `asyncio.to_thread`)
- Input bounds validation in the worker itself (see 2.4)

### 6.3 Unbounded Background Goroutines/Threads

- Cleanup/timer loops spawned without a context or stop channel (leak per construction; matters in tests and re-instantiation)

### 6.4 Auth Hot-Path Database Load (DoS amplifier)

- Columns queried on **every authenticated request** (API-key hash lookup, session lookup) must be indexed — `WHERE key_hash = $1` with no index = sequential scan per request
- Synchronous per-request writes on the auth path (`UPDATE last_used_at = now()`) — suggest throttling/batching

---

## Part 7: Frontend / Client-Side Security

Applicable when the repo contains a browser client (React/Vue/Svelte/plain JS).

- **Token storage:** JWT/session tokens in `localStorage`/`sessionStorage` = exfiltratable by any XSS. Prefer `httpOnly; Secure; SameSite` cookies + CSRF token. If localStorage is kept, verify short TTL + refresh-token rotation is actually wired up (an unused `refresh()` helper is a finding).
- **Tokens in URLs** from the client side (see 4.3) — grep for `?token=`, `&access_token=`, EventSource URLs
- **XSS surface:** see 3.3 (`dangerouslySetInnerHTML`, href schemes); also `window.open` without `noopener`, `postMessage` handlers without origin checks
- **Open redirect:** post-auth redirect helpers must reject absolute/protocol-relative URLs (allow only same-origin paths starting `/` and not `//`)
- **Auth error handling:** interceptors that force-logout/hard-reload on 401s from non-session endpoints (check the endpoint allow-list covers ALL auth provider routes, not just some)
- Note as positive when present: strict CSP compatibility, no `eval`, server-validated OAuth state, JSX auto-escaping relied on (no string HTML assembly)

---

## Part 8: Supply Chain and CI/CD (SSDF RV.1/PW.4, SLSA, OWASP A06/A08)

These are **absence checks** — grep proves nothing exists; verify each control:

### 8.1 Dependency Management

- [ ] Automated dependency updates configured (`.github/dependabot.yml` or `renovate.json`) covering ALL ecosystems present (gomod, npm — each workspace, pip/uv, docker, github-actions)
- [ ] Lockfiles committed (`go.sum`, `package-lock.json`, `uv.lock`/`poetry.lock`) and CI installs with frozen/ci mode (`npm ci`, `uv sync --frozen`, `go mod download` from committed sum)

### 8.2 Vulnerability and Secret Scanning in CI

- [ ] Dependency/CVE scanning (Trivy fs, grype, `govulncheck`, `npm audit` gate, `pip-audit`) — grep workflows for `trivy|grype|codeql|govulncheck|snyk`; zero hits = finding
- [ ] Container image scanning before publish
- [ ] Static analysis (CodeQL/Semgrep) for each major language
- [ ] Secret scanning on PRs (gitleaks/trufflehog) or org-level GitHub secret scanning confirmed
- [ ] SBOM generation for published images (`anchore/sbom-action`, `docker buildx --sbom`)

### 8.3 Workflow Integrity

- [ ] Third-party actions pinned to full 40-char commit SHAs, not mutable tags (`uses: foo/bar@v1` can be repointed to malicious code running with `GITHUB_TOKEN`). Prioritize low-reputation actions with high-privilege tokens (`packages: write`, `contents: write`).
- [ ] Workflow `permissions:` blocks set to least privilege; no `pull_request_target` with checkout of untrusted code
- [ ] No secrets exposed to fork PRs

---

## Part 9: Container and Infrastructure Hardening (CIS Docker, OWASP A05)

### 9.1 Dockerfiles

- [ ] `USER` directive — final stage must run as non-root (static Go binaries never need root; use `nginxinc/nginx-unprivileged` for nginx)
- [ ] Base images pinned by digest (`image@sha256:...`), no `:latest` anywhere (Dockerfiles AND compose files)
- [ ] `.dockerignore` exists for every build context — check what the context actually is in compose/build commands; a repo-root context without `.dockerignore` ships `.env`, `.git/`, and data dirs to the daemon, one `COPY` mistake from a leaked layer
- [ ] `HEALTHCHECK` in images (and compose healthchecks for every service, including workers)
- [ ] No secrets in `ENV`/`ARG`/layers

### 9.2 Compose / Orchestration

- [ ] `cap_drop: [ALL]`, `security_opt: [no-new-privileges:true]`, `read_only: true` where feasible
- [ ] Internal services (DB, queue, object store, health/metrics ports) not exposed on host/all interfaces
- [ ] Secrets via env/secret mounts, not inline values

### 9.3 Reverse Proxy (nginx/Caddy/Traefik)

- [ ] CSP and HSTS present, plus `X-Content-Type-Options`, `X-Frame-Options`/`frame-ancestors`, `Referrer-Policy`, `server_tokens off`
- [ ] **nginx inheritance trap:** `add_header` directives are NOT inherited into `location` blocks that declare their own `add_header` — headers defined only at `server` level are silently dropped on proxied/static responses. Verify headers apply to every location (use an included snippet). Check with the proxy's real semantics, not by header presence in the file.
- [ ] Proxy-layer rate limiting (`limit_req_zone`/`limit_conn`) on API locations, with burst allowance for SSE/streaming
- [ ] Request body size limits at the proxy

### 9.4 Host / Deploy

- [ ] systemd units: dedicated non-root `User=`, `NoNewPrivileges=`, `ProtectSystem=`, `PrivateTmp=`; no placeholder paths left unfilled
- [ ] Install/deploy scripts don't print secrets (see 1.7)
- [ ] IaC (Terraform/K8s): no plaintext secrets, least-privilege roles, no public buckets/SGs unless intended

---

## Scan Procedure

1. **Detect the stack** (languages, frameworks, infra surfaces) and decide which parts apply; note what's skipped.
2. **Choose mode** — full vs staged/diff (state the diff-scan limitation in the report).
3. **Run Part 1** (secrets) — always, in every mode.
4. **Run Parts 2–7** against application code — backend, frontend, workers, real-time. Trace auth flows end-to-end (Principle 2); inventory missing controls (Principle 1).
5. **Run Parts 8–9** against CI/CD and infrastructure files (full scans; in diff scans, when those files are touched).
6. **Cross-reference** findings: an endpoint missing auth that also lacks rate limiting is worse than either alone; a spoofable rate-limiter key downgrades every rate-limit control.
7. **Verify** the highest-severity claims by reading the code path; mark them verified.
8. **Prioritize** by severity and exploitability.

---

## Report Format

```markdown
## Security Scan Complete - [SAFE/UNSAFE]

### Scan Scope
- Mode: full / staged-diff (if diff: note limitation + recommend periodic full scan)
- Stack detected: <e.g. Go API, React SPA, Python worker, Docker/nginx, GitHub Actions>
- Files scanned: N
- Not applicable (skipped): <section — reason>

---

### Critical Findings
(Must fix before commit/deploy)

| # | Category | File | Finding | Severity | CWE/OWASP/Framework |
|---|----------|------|---------|----------|---------------------|
| 1 | Auth Bypass | [handler/foo.go:42](handler/foo.go#L42) | Endpoint missing auth middleware (verified) | Critical | API2:2023 |

### High Findings
(Should fix soon)

### Medium / Low Findings
(Address in next iteration)

---

### Secrets Audit
- .env ignored: YES/NO
- Hardcoded secrets found: YES/NO
- Environment variables used correctly: YES/NO
- Scripts/CI leak secrets to output: YES/NO

### Control Inventory
- [ ] Object/topic-level authorization incl. real-time (API1)
- [ ] Authentication, OAuth state, revocation, trusted proxies (API2)
- [ ] Property-level authorization (API3)
- [ ] Resource limits: API + workers + connections (API4)
- [ ] Function-level authorization (API5)
- [ ] Business flow abuse (API6)
- [ ] SSRF protections (API7)
- [ ] Security configuration + headers actually served (API8)
- [ ] API inventory (API9)
- [ ] Third-party API safety (API10)
- [ ] CRLF/header injection (email, HTTP, logs)
- [ ] No tokens in URLs
- [ ] Worker crash-safety, ack deadlines, poison messages
- [ ] Frontend token storage + XSS surface
- [ ] Dependency updates + CVE/secret scanning + SBOM in CI
- [ ] CI actions SHA-pinned, least-privilege tokens
- [ ] Containers non-root, images digest-pinned, .dockerignore
- [ ] Proxy: CSP/HSTS served on all locations, proxy rate limits

---

**Status**: [SAFE/UNSAFE] — N critical, N high, N medium, N low findings
```

## Severity Classification

| Severity | Criteria |
|----------|----------|
| **Critical** | Direct exploitation possible: auth bypass, SQL injection, hardcoded secrets, RCE |
| **High** | Significant risk with some conditions: IDOR/topic BOLA, mass assignment, SSRF, rate-limit bypass via spoofed headers, OAuth state replay, weak crypto for auth, mutable-tag actions with write tokens, worker crash on poison input |
| **Medium** | Moderate risk or defense-in-depth gap: missing rate limits, no revocation, verbose errors, missing headers, root containers, no dependency scanning, tokens in URLs |
| **Low** | Best practice violation, minimal direct risk: info disclosure in headers, missing CSP on low-risk app, missing healthchecks, image tag pinning |

## Example Scans

```
/security-scan                       # full scan, all applicable parts
/security-scan src/                  # specific directory
/security-scan --staged              # staged files (pre-commit; states diff limitation)
/security-scan --api                 # Parts 2-5 only
/security-scan --infra               # Parts 8-9 only (supply chain + containers/proxy)
```

## Notes

- Read files with the Read tool for accurate line numbers; use Grep for sweeps
- Provide clickable file references: `[file.go:42](file.go#L42)`
- Reference OWASP/CWE/SSDF/SLSA/CIS identifiers per finding; include a concrete remediation for each
- Flag suspicious patterns even when uncertain — but distinguish "verified" from "needs verification"
- Report notable *positive* controls too (parameterized queries throughout, correct AEAD usage, fail-fast secret validation) — it calibrates trust in the rest of the report
