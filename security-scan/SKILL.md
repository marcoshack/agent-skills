---
name: security-scan
description: Scan the codebase for leaked secrets, passwords, API keys, and sensitive information before committing to git. Use when user asks to check for secrets, scan for security issues before commit, or audit sensitive data.
disable-model-invocation: false
allowed-tools: Grep, Glob, Read, Bash
---

# Security Scan

Perform a comprehensive security audit of all non-ignored files to detect leaked secrets and sensitive information before committing to the repository.

## Scan Procedure

### 1. Identify Files to Scan

Use git to determine which files are tracked and would be committed:
```bash
# List all tracked files
git ls-files

# List untracked files not in .gitignore
git ls-files --others --exclude-standard

# Check current git status
git status --porcelain
```

### 2. Read .gitignore Configuration

Read `.gitignore` to verify sensitive files are properly excluded:
- `.env` files containing actual secrets
- Credential files
- Private keys
- Build artifacts that might contain secrets

### 3. Scan for Common Secret Patterns

Search for hardcoded secrets using pattern matching:

**API Keys and Tokens:**
- `ANTHROPIC_API_KEY`
- `OPENAI_API_KEY`
- `DISCORD_WEBHOOK`
- `GITHUB_TOKEN`
- `AWS_SECRET_ACCESS_KEY`
- Any pattern like `api[_-]?key.*=.*[a-zA-Z0-9]{20,}`

**Credentials:**
- Database passwords
- OpenSearch/Elasticsearch credentials
- Discord webhooks
- OAuth secrets
- JWT secrets

**Sensitive Data:**
- Private SSH keys
- TLS/SSL certificates
- Email addresses in config files
- URLs with embedded credentials (`https://user:pass@host`)

### 4. Verify Environment Variable Usage

Check that all sensitive configuration uses environment variables:
- Code should use `os.environ.get()`, `os.getenv()`, or `process.env`
- Docker Compose should use `${VAR}` substitution
- No hardcoded values for secrets

### 5. Check Staged Changes

If files are staged for commit:
```bash
git diff --cached
```

Ensure no secrets are about to be committed.

### 6. Pattern-Based Secret Detection

Use grep to find potential secrets in tracked files:
```bash
git ls-files | xargs grep -iE "(api[_-]?key|password|secret|token|webhook).*[=:].*['\"]?[a-zA-Z0-9]{20,}"
```

### 7. Generate Security Report

Provide a clear, actionable report with:

**Safe to Commit ✅**
- List all scanned files
- Confirm no hardcoded secrets found
- Verify .env files are ignored
- Confirm environment variables used correctly

**Security Issues Found ⚠️**
- File path and line number
- Type of secret/issue
- Severity (Critical, High, Medium, Low)
- Recommended fix

## Critical Checks

1. **Actual .env file must be ignored**
   - Verify `.env` exists but is in `.gitignore`
   - Confirm `.env` is not tracked by git: `git check-ignore -v .env`

2. **Template files should be empty**
   - `.env.example` should contain only placeholder values
   - No actual API keys in example files

3. **No credentials in code**
   - All API keys read from environment
   - No hardcoded passwords
   - No webhook URLs in source code

4. **No credentials in Docker/CI files**
   - Docker Compose uses environment variable substitution
   - Dockerfiles don't contain secrets
   - CI/CD configs use secret management

## Report Format

Structure the final report as:

```markdown
## 🔒 Security Scan Complete - [SAFE/UNSAFE] to Commit

### Files Scanned:
- file1.py ✅
- file2.yml ✅
- ...

### Security Findings:

✅ NO SECRETS DETECTED (if clean)
or
⚠️ SECRETS DETECTED (if issues found)

### Protected Data:
- API_KEY_NAME - ✅ Environment variable
- ...

### Issues (if any):
- [file.py:42](file.py#L42) - Critical: Hardcoded API key
- ...

---

**Status**: Repository is [secure/INSECURE] and [ready/NOT READY] to commit.
```

## Example Scans

**Checking specific directories:**
```
/security-scan src/
```

**Full repository scan:**
```
/security-scan
```

**Before committing:**
```
/security-scan --staged
```

## Notes

- Focus on files tracked by git (or about to be committed)
- Read files using Read tool for accurate line numbers
- Use Grep for pattern matching across multiple files
- Provide clickable file references: `[file.py:42](file.py#L42)`
- Be concise but thorough
- Always err on the side of caution - flag suspicious patterns
