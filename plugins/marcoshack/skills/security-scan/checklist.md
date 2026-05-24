# Security Scanning Checklist

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
