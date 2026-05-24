---
name: commit
description: Create a well-structured git commit with security scanning, change analysis, and conventional commit messages. Use when the user asks to commit, create a commit, or save their changes.
disable-model-invocation: false
allowed-tools: Bash, Read, Grep, Glob, Skill
user-invocable: true
---

# Commit

Create a secure, well-structured git commit by analyzing all changes, running a security scan, and writing a clear commit message.

## Procedure

### 1. Gather Repository State

Run these commands in parallel to understand the current state:

```bash
# See all modified, staged, and untracked files (never use -uall)
git status --porcelain

# See the full diff of all changes (staged + unstaged)
git diff HEAD

# See untracked files that would be added
git ls-files --others --exclude-standard

# Recent commit messages to match the repository's style
git log --oneline -10
```

### 2. Run Security Scan

Before proceeding, ask the user whether they want to run a security scan:

- **Run security scan (Recommended)**: Invoke the `/security-scan` skill to check for leaked secrets, credentials, or sensitive data in the changes.
- **Skip security scan**: Skip the scan and proceed directly to analyzing changes.

**If the security scan runs and finds issues**: STOP immediately. Report the findings to the user and do NOT create a commit. Wait for the user to resolve the issues before retrying.

**If the security scan passes or was skipped**: Continue to the next step.

### 3. Analyze Changes

Read and understand all changes that will be committed:

- Review the diff output carefully
- Identify the purpose of each change (new feature, bug fix, refactor, docs, config, etc.)
- Group related changes together
- Note any files that should NOT be committed (temporary files, local config, generated artifacts)

**Do not commit files that likely contain secrets** (`.env`, `credentials.json`, private keys, etc.) even if they are not in `.gitignore`. Warn the user if such files are present in the changes.

### 4. Stage Files

Stage files intentionally. Prefer adding specific files by name rather than `git add -A` or `git add .`:

```bash
git add file1.py file2.py ...
```

Only use `git add -A` if the user explicitly requests it or if all changed files clearly belong in the commit.

If there are unrelated changes that should be separate commits, ask the user whether they want to commit everything together or split into multiple commits.

### 5. Write Commit Message

Write a commit message following these rules:

**Title line (first line):**
- Maximum 72 characters
- Use imperative mood ("Add feature" not "Added feature" or "Adds feature")
- Summarize the single most important change or the overall theme
- Do NOT use a conventional commit prefix (e.g., `feat:`, `fix:`) unless the repository already uses them (check `git log`)
- Capitalize the first word
- No period at the end

**Body (separated by a blank line):**
- Wrap lines at 72 characters
- Explain WHAT changed and WHY, not HOW (the diff shows how)
- Use bullet points for multiple changes
- Group related changes under short descriptions
- Omit trivial details (formatting, import reordering) unless that's the point of the commit

**Example:**
```
Add agent session persistence across restarts

- Introduce FileSessionManager to save conversation history to disk
- Add SNAKE_SESSIONS_DIR env var for configurable storage path
- Mount sessions directory as Docker volume for container survival
- Update agent runner to reuse existing sessions on startup
```

### 6. Create the Commit

Always pass the commit message via a HEREDOC for correct formatting:

```bash
git commit -m "$(cat <<'EOF'
Title line here

Body line 1
Body line 2
EOF
)"
```

**IMPORTANT**: Do NOT add a `Co-Authored-By` trailer to the commit message.

### 7. Verify

After committing, run `git status` to confirm the commit succeeded and the working tree is in the expected state. Show the user the commit hash and title.

## Rules

- **Never amend** a previous commit unless the user explicitly asks for it
- **Never force push** unless the user explicitly asks for it
- **Never push** to a remote unless the user explicitly asks for it
- **Never skip hooks** (no `--no-verify`) unless the user explicitly asks for it
- **Never use interactive flags** (`-i`) as they require interactive input
- If a pre-commit hook fails, fix the issue, re-stage, and create a NEW commit (do not amend)
- If there are no changes to commit, inform the user instead of creating an empty commit
- If the user provides a specific commit message, use it as-is
