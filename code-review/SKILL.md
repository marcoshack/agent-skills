---
name: code-review
description: Review code for software engineering best practices, idiomatic patterns, security risks, performance, error handling, testing gaps, concurrency issues, API contracts, and dependency concerns. Focuses on changed files by default. Use when the user asks to review code, review changes, or check code quality.
disable-model-invocation: false
allowed-tools: Bash, Read, Grep, Glob, Skill
user-invocable: true
---

# Code Review

Perform a thorough code review focusing on software engineering best practices, idiomatic code, security, performance, and correctness.

## Procedure

### 1. Determine Review Scope

By default, review only **changed files**. Only review the full workspace or specific files if the user explicitly requests it.

**If the workspace has git:**

```bash
# Check if we're in a git repo
git rev-parse --is-inside-work-tree

# Get the list of changed files (staged + unstaged + untracked)
git status --porcelain

# Get the full diff of changes against HEAD
git diff HEAD

# If on a feature branch, also diff against the base branch
git log --oneline -5
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

If a merge base is found, also run:
```bash
git diff $(git merge-base HEAD main)..HEAD
```

Use the diff output as the primary review target. Read full files only when surrounding context is needed to understand the change.

**If no git is available:**

Ask the user which files to review. If they say "all" or the workspace is small, scan the workspace with `Glob` and review all source files.

### 2. Read Changed Files for Context

For each changed file, read the full file (or relevant sections) to understand:
- The broader context around the changed lines
- Imports and dependencies used
- The module/class/function the change lives in
- Related code that may be affected by the change

### 3. Perform the Review

Evaluate changes across all of the following areas. Skip areas that are not applicable to the changes under review.

---

#### 3.1 Software Engineering Best Practices

**Check for:**
- **Naming**: Are variables, functions, classes, and files named clearly and consistently? Do names convey intent?
- **Single Responsibility**: Does each function/class/module do one thing well? Are functions excessively long or doing too much?
- **DRY violations**: Is there duplicated logic that should be extracted? (But don't flag cases where a small amount of duplication is clearer than an abstraction.)
- **SOLID principles**: Open/closed violations, interface segregation issues, dependency inversion problems — flag only when clearly harmful, not pedantically.
- **Readability**: Could someone unfamiliar with the code understand it? Are there confusing control flows, deeply nested conditions, or magic numbers?
- **Code organization**: Are changes in the right file/module? Does the change follow the existing project structure?
- **Dead code**: Unused variables, unreachable branches, commented-out code left behind.
- **Consistent style**: Does the change match the surrounding code style (even if the reviewer might prefer a different style)?

#### 3.2 Idiomatic Code per Language

Adapt the review to the language being used. Examples (non-exhaustive):

**Python:**
- Use list comprehensions over manual loops where appropriate
- Prefer `pathlib` over `os.path` for path manipulation
- Use context managers (`with`) for resource management
- Type hints on public function signatures
- Avoid mutable default arguments
- Use `enumerate()` instead of manual index tracking
- Prefer `f-strings` over `%` or `.format()` for string formatting

**JavaScript/TypeScript:**
- Prefer `const` over `let`; avoid `var`
- Use optional chaining (`?.`) and nullish coalescing (`??`) where appropriate
- Avoid `any` type in TypeScript — prefer proper typing or `unknown`
- Use async/await over raw Promises or callbacks
- Destructuring where it improves clarity
- Avoid `==` (use `===`)
- Proper module imports (named vs default, barrel files)

**Go:**
- Return errors, don't panic for expected failures
- Use `errors.Is` / `errors.As` for error checking
- Prefer table-driven tests
- Avoid `init()` functions when possible
- Use meaningful receiver names (not single letters unless idiomatic)
- Context propagation through function chains
- Proper use of goroutines and channels

**Rust:**
- Prefer `?` operator over manual `match` for error propagation
- Use `clippy` lint suggestions as guidelines
- Prefer iterators and combinators over manual loops
- Proper ownership and borrowing patterns
- Use `derive` macros appropriately
- Avoid unnecessary `clone()` and `unwrap()`

**Java/Kotlin:**
- Use streams and lambdas where appropriate
- Proper use of Optional (Java) / nullability (Kotlin)
- Follow the project's existing patterns for DI
- Immutability by default
- Proper exception hierarchy

**For other languages**: Apply the same principle — identify the idiomatic way to express the logic and flag deviations that hurt readability or correctness.

#### 3.3 Security Risks (Lightweight)

Perform a local, language-level security review of the changed code. This is NOT a full security audit — flag obvious issues and recommend `/security-scan` for deeper analysis.

**Check for:**
- **Input validation**: Is user input validated/sanitized before use? SQL queries parameterized?
- **Injection risks**: String interpolation in SQL, shell commands, or HTML rendering with user data
- **Hardcoded secrets**: API keys, passwords, tokens visible in the diff
- **Unsafe deserialization**: Deserializing untrusted data without validation
- **Path traversal**: User input used in file paths without sanitization
- **Insecure randomness**: Using non-cryptographic RNG for security-sensitive values
- **Sensitive data exposure**: Logging PII, returning internal errors to clients, credentials in URLs

If ANY security finding seems moderate or higher severity, add a recommendation at the end of the review:

> **Security note**: This review identified potential security concerns. Consider running `/security-scan` for a comprehensive security audit before merging.

#### 3.4 Performance & Complexity

**Check for:**
- **N+1 queries**: Database calls inside loops, repeated fetches that could be batched
- **Unnecessary allocations**: Creating objects/arrays in loops when they could be reused or pre-allocated
- **Algorithmic complexity**: O(n^2) or worse in code paths that handle potentially large inputs
- **Missing pagination/limits**: Unbounded queries or collection processing
- **Blocking operations**: Synchronous I/O in async contexts, blocking the event loop
- **Unnecessary work**: Repeated calculations, redundant API calls, fetching data that isn't used
- **Memory leaks**: Event listeners not cleaned up, growing caches without eviction, unclosed resources

Only flag performance issues that are **likely to matter** given the context. Don't micro-optimize code that runs once at startup or handles small datasets.

#### 3.5 Error Handling & Edge Cases

**Check for:**
- **Swallowed errors**: Empty `catch` blocks, ignored return values from fallible operations, `_ = err`
- **Missing error propagation**: Errors caught but not logged, re-thrown, or handled meaningfully
- **Null/undefined guards**: Accessing properties on potentially null values without checks
- **Incomplete pattern matching**: `switch`/`match` without exhaustive cases or default handling
- **Boundary conditions**: Off-by-one errors, empty collections, zero values, negative numbers
- **Resource cleanup**: Missing `finally`/`defer`/`with` for resources that need closing (files, connections, locks)
- **Partial failure**: What happens if a multi-step operation fails midway? Is state left consistent?

#### 3.6 Testing Impact

**Check for:**
- **Changed logic without test changes**: If business logic or branching was modified, were tests updated or added?
- **New code paths untested**: New branches, error cases, or edge cases introduced without coverage
- **Test quality**: Are new/modified tests actually asserting meaningful behavior? Do they test the right things?
- **Flaky test patterns**: Tests that depend on timing, external services, or non-deterministic ordering

Do NOT demand 100% coverage. Focus on:
- Critical business logic must be tested
- New error handling paths should be tested
- Edge cases identified in 3.5 should have test cases

If tests are clearly missing for significant logic changes, suggest specific test cases.

#### 3.7 Concurrency Issues

**Check for:**
- **Race conditions**: Shared mutable state accessed from multiple threads/goroutines/async tasks without synchronization
- **Deadlocks**: Lock ordering violations, holding locks across await points
- **Improper async/await**: Missing `await` on promises, blocking in async functions, fire-and-forget without error handling
- **Thread safety**: Mutable globals, non-atomic read-modify-write sequences
- **Resource contention**: Unbounded goroutine/thread spawning, missing backpressure

Skip this section entirely if the changes don't involve concurrent code.

#### 3.8 API Contract & Breaking Changes

**Check for:**
- **Changed function signatures**: Parameters added/removed/reordered on public/exported functions
- **Changed return types**: Different return shape, removed fields, changed nullability
- **API response changes**: Modified JSON response structure, removed fields, changed status codes
- **Config/schema changes**: Modified config file format, database schema changes, environment variable changes
- **Changed behavior**: Same interface but different semantics (e.g., function now throws where it didn't before)
- **Protocol changes**: Modified message formats, changed event names, altered middleware ordering

For each breaking change found, flag it clearly and ask whether it's intentional and whether all consumers have been updated.

#### 3.9 Documentation Gaps

Only flag documentation issues for:
- **Public APIs / exported functions** where the behavior, parameters, or return value are non-obvious
- **Complex algorithms** where a brief comment explaining the approach would help
- **Non-obvious side effects** that callers need to know about
- **Configuration** that users/operators need to understand

Do NOT flag:
- Missing docs on private/internal functions with clear names
- Missing docs on simple getters/setters/CRUD operations
- Missing file-level or module-level comments unless the module's purpose is unclear

#### 3.10 Dependency Concerns

**Check for:**
- **New dependencies added**: Are they necessary? Could the functionality be achieved with existing deps or standard library?
- **Dependency size**: Is a large library pulled in for a small feature? (e.g., adding lodash for a single utility)
- **Maintenance status**: Is the dependency actively maintained? (Don't look this up — flag if you recognize it as abandoned/deprecated)
- **Duplicate functionality**: Does the project already have a dependency that does the same thing?
- **Version pinning**: Are new dependencies pinned to specific versions?

---

### 4. Compile the Review

Structure the review report as follows:

```markdown
## Code Review - [PASS / CHANGES REQUESTED]

### Scope
- Files reviewed: N
- Review mode: Changed files (git diff) / Full workspace / Specific files

---

### Critical Issues
(Must fix before merge — correctness bugs, security risks, data loss potential)

| # | Area | File | Finding | Suggestion |
|---|------|------|---------|------------|
| 1 | Security | [file.py:42](file.py#L42) | SQL query built with string concatenation | Use parameterized queries |

### Improvements
(Should fix — best practice violations, performance, missing error handling)

| # | Area | File | Finding | Suggestion |
|---|------|------|---------|------------|
| 1 | Error Handling | [file.py:88](file.py#L88) | Exception caught but not logged | Log the error or propagate it |

### Suggestions
(Nice to have — style, idioms, minor enhancements)

| # | Area | File | Finding | Suggestion |
|---|------|------|---------|------------|
| 1 | Idiomatic | [file.py:15](file.py#L15) | Manual loop building a list | Use list comprehension |

### Testing
- [ ] Test coverage adequate for changes: YES/NO
- Suggested test cases (if any):
  - ...

### Breaking Changes
- [ ] Public API changes: YES/NO
- [ ] Config/schema changes: YES/NO
- Details (if any): ...

---

**Verdict**: [PASS / CHANGES REQUESTED] — N critical, N improvements, N suggestions
```

### 5. Security Escalation

If the review uncovered security concerns (even low severity), append:

> **Recommendation**: Run `/security-scan` for a comprehensive security audit covering OWASP API Top 10, secret detection, and injection vulnerabilities.

---

## Rules

- **Be constructive**: Every criticism must come with a concrete suggestion or example fix
- **Respect existing patterns**: Don't suggest rewriting code to match your preference if the project has an established convention
- **Prioritize ruthlessly**: Critical issues first, style nits last. Don't bury important findings in a long list of minor suggestions
- **Avoid false positives**: If you're unsure whether something is an issue, say so rather than presenting it as a definite problem
- **Don't review generated code**: Skip auto-generated files, lock files, vendor directories, compiled output
- **Stay scoped**: Review only what was asked. Don't expand into reviewing the entire codebase unless requested
- **Be concise**: Explain issues in 1-2 sentences. Link to the exact line. Don't write essays
- **Acknowledge good changes**: If the code is well-written, say so briefly. Not everything needs criticism

## Example Usage

**Review current changes (default):**
```
/code-review
```

**Review a specific file:**
```
/code-review src/auth/login.ts
```

**Review a PR branch against main:**
```
/code-review --base main
```

**Review with focus on a specific area:**
```
/code-review --focus security
/code-review --focus performance
```
