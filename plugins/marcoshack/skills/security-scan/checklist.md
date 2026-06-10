# Security Scanning Checklist

Part A is the secret-detection pattern reference. Part B is the control-inventory checklist — controls that must be verified to EXIST (grep cannot find an absence; check each one deliberately).

# Part A: Secrets Detection

## Secrets Detection Patterns

### API Keys and Authentication Tokens
- [ ] `ANTHROPIC_API_KEY`
- [ ] `OPENAI_API_KEY`
- [ ] `OPENAI_ORGANIZATION`
- [ ] `GITHUB_TOKEN`
- [ ] `GITLAB_TOKEN`
- [ ] `AWS_ACCESS_KEY_ID`
- [ ] `AWS_SECRET_ACCESS_KEY`
- [ ] `AZURE_CLIENT_SECRET`
- [ ] `GCP_SERVICE_ACCOUNT_KEY`

### Database Credentials
- [ ] `DATABASE_URL` with embedded credentials
- [ ] `DB_PASSWORD`
- [ ] `MYSQL_PASSWORD`
- [ ] `POSTGRES_PASSWORD`
- [ ] `MONGODB_PASSWORD`
- [ ] `REDIS_PASSWORD`
- [ ] `OPENSEARCH_PASSWORD`
- [ ] `ELASTICSEARCH_PASSWORD`

### Webhooks and Integrations
- [ ] `DISCORD_WEBHOOK`
- [ ] `SLACK_WEBHOOK`
- [ ] `WEBHOOK_URL`
- [ ] Discord webhook URLs: `https://discord.com/api/webhooks/...`
- [ ] Slack webhook URLs: `https://hooks.slack.com/services/...`

### Cryptographic Material
- [ ] Private keys (`-----BEGIN PRIVATE KEY-----`)
- [ ] RSA keys (`-----BEGIN RSA PRIVATE KEY-----`)
- [ ] SSH keys in non-standard locations
- [ ] JWT secrets (`JWT_SECRET`)
- [ ] Encryption keys
- [ ] Certificate files (`.pem`, `.key`, `.p12`)

### OAuth and Session Tokens
- [ ] `CLIENT_SECRET`
- [ ] `SESSION_SECRET`
- [ ] `OAUTH_TOKEN`
- [ ] `REFRESH_TOKEN`
- [ ] `ACCESS_TOKEN`

## File-Specific Checks

### Configuration Files
- [ ] `.env` - Must be in `.gitignore`
- [ ] `.env.local`, `.env.production` - Should be ignored
- [ ] `config.json`, `secrets.json` - Should not contain real secrets
- [ ] `credentials.json` - Should be ignored
- [ ] `.env.example` - Should only contain placeholders

### Code Files
- [ ] No hardcoded API keys in Python/JS/etc.
- [ ] No passwords in connection strings
- [ ] No tokens in HTTP headers (unless from env)
- [ ] No credentials in SQL queries

### Docker and CI/CD
- [ ] `Dockerfile` - No secrets in ENV or ARG
- [ ] `docker-compose.yml` - Uses `${VAR}` substitution
- [ ] `.github/workflows/*.yml` - Uses secrets context
- [ ] `Makefile` - No hardcoded credentials

### Cloud and Infrastructure
- [ ] Terraform files - No secrets in `.tf` files
- [ ] Kubernetes manifests - No secrets in YAML
- [ ] Ansible playbooks - Uses vault or env vars

## Regex Patterns for Detection

```regex
# Generic secret patterns
(api[_-]?key|password|secret|token|webhook).*[=:].*["']?[a-zA-Z0-9]{20,}

# API keys (various formats)
[a-zA-Z0-9_-]{32,}

# AWS access keys
AKIA[0-9A-Z]{16}

# Generic tokens
[a-zA-Z0-9_-]{40,}

# Discord webhooks
discord\.com/api/webhooks/[0-9]+/[a-zA-Z0-9_-]+

# Slack webhooks
hooks\.slack\.com/services/[A-Z0-9]+/[A-Z0-9]+/[a-zA-Z0-9]+

# Private keys
-----BEGIN.*PRIVATE KEY-----

# Email addresses in config (sometimes sensitive)
[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}

# URLs with embedded credentials
[a-z]+://[^:]+:[^@]+@
```

## Best Practices Validation

### Environment Variables
- [ ] All secrets loaded from environment
- [ ] `.env` file exists and is ignored
- [ ] `.env.example` provided with empty values
- [ ] Code uses `os.getenv()` or similar

### Git Configuration
- [ ] `.gitignore` includes `.env`
- [ ] `.gitignore` includes common secret files
- [ ] No secrets in git history (use `git log -p --all`)
- [ ] `.env` properly ignored: `git check-ignore .env`

### Documentation
- [ ] README doesn't contain real credentials
- [ ] Setup instructions reference .env.example
- [ ] No secrets in code comments

### Docker Security
- [ ] Secrets passed via environment variables
- [ ] No secrets in image layers
- [ ] `.dockerignore` excludes `.env`
- [ ] Multi-stage builds don't leak secrets

## Severity Guidelines

**Critical** - Immediate action required
- Live API keys or tokens in code
- Database credentials in files
- Private keys committed

**High** - Should be fixed before merge
- Webhook URLs in code
- URLs with embedded credentials
- Secrets in config files

**Medium** - Should be addressed
- Missing .gitignore entries
- Secrets in comments
- Development credentials in code

**Low** - Best practice improvements
- .env.example improvements
- Documentation clarifications
- Non-sensitive environment variables

## Remediation Steps

If secrets are found:

1. **Remove from code**
   - Replace with environment variable
   - Update .gitignore
   - Remove from git history if committed

2. **Rotate compromised secrets**
   - Generate new API keys
   - Update passwords
   - Invalidate old tokens

3. **Prevent future leaks**
   - Add pre-commit hooks
   - Use secret scanning tools
   - Regular security audits

---

# Part B: Control Inventory (absence checks)

For each item, verify the control exists. Skip items not applicable to the detected stack and note them as skipped in the report.

## Authentication & Session Controls

- [ ] Token revocation exists (logout invalidates server-side; `tokens_valid_after` or `jti` denylist)
- [ ] Disabled/demoted users lose access before token expiry (per-request active check OR short TTL + refresh)
- [ ] All credential types have parity (e.g. API keys re-check `IsActive` → JWTs must too)
- [ ] OAuth `state` is a per-session nonce bound via cookie, exact-matched at callback (not just server-signed)
- [ ] PKCE on public OAuth clients; exact redirect URI matching
- [ ] Rate limiter on login/register/reset — keyed on a non-spoofable identity (trusted-proxy config for `X-Forwarded-For`)
- [ ] Tokens never travel in URL query strings (watch SSE/EventSource, download links)
- [ ] Frontend stores tokens in httpOnly cookies, or has documented XSS mitigations + short TTL + working refresh

## Authorization Controls

- [ ] Every resource-by-ID endpoint checks ownership/tenancy
- [ ] Real-time topic/channel subscriptions are authorized per topic, not just per connection
- [ ] Admin routes behind role middleware (not just in-handler checks)
- [ ] Public/share endpoints minimize PII in their payloads

## Availability / Worker Controls

- [ ] Worker pools / queue consumers have panic-or-exception recovery (one poison task ≠ process crash)
- [ ] Ack deadlines exceed worst-case task runtime (or tasks heartbeat)
- [ ] Non-retryable errors are terminated + reported, not NAK-retried forever
- [ ] Workers validate input bounds (max sizes/counts) even from trusted internal callers
- [ ] CPU-bound work doesn't block async event loops / health endpoints
- [ ] SSE/WebSocket hubs cap connections per user; channel close paths are idempotent
- [ ] Background loops accept a context/stop channel
- [ ] Auth hot-path DB columns (key hash, session lookups) are indexed

## Supply Chain & CI/CD

- [ ] Dependabot/Renovate covers every ecosystem (gomod, npm per workspace, pip/uv, docker, github-actions)
- [ ] Lockfiles committed; CI uses frozen installs (`npm ci`, `uv sync --frozen`)
- [ ] CVE scanning in CI (Trivy/grype/govulncheck/CodeQL) — grep workflows; zero hits = finding
- [ ] Secret scanning on PRs (gitleaks/trufflehog or org-level)
- [ ] SBOM generated for published images
- [ ] Third-party actions pinned to 40-char SHAs (prioritize ones with `packages: write`/`contents: write`)
- [ ] Workflow `permissions:` least-privilege; no `pull_request_target` + untrusted checkout

## Containers & Infrastructure

- [ ] Every Dockerfile final stage has a non-root `USER`
- [ ] Base images digest-pinned; no `:latest` in Dockerfiles or compose
- [ ] `.dockerignore` exists for every build context (check actual context paths in compose)
- [ ] `HEALTHCHECK` in images; compose healthchecks for all services including workers
- [ ] Compose hardening: `cap_drop: [ALL]`, `no-new-privileges`, `read_only` where feasible
- [ ] Internal ports (DB, queue, health, metrics) not bound to `0.0.0.0`/host
- [ ] Proxy serves CSP + HSTS + nosniff + frame protection on ALL locations (mind nginx `add_header` inheritance)
- [ ] Proxy-layer rate limiting and body-size limits
- [ ] systemd units: non-root `User=`, `NoNewPrivileges=`, `ProtectSystem=`, `PrivateTmp=`
- [ ] Install/deploy scripts never echo secrets
