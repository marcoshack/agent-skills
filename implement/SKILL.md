---
name: implement
description: Implement a Taskwondo feature or fix for a given work item. Use when the user says "work on PROJ-123" or "implement PROJ-123" or references a work item display ID.
argument-hint: <DISPLAY_ID> [description]
---

# Implement Work Item

You are implementing a feature or fix for the Taskwondo project. The user has provided a work item display ID (e.g. `PROJ-123`) and optionally a description of what to do.

**Arguments:** `$ARGUMENTS`

## 1. Parse the Work Item

Extract the display ID from the arguments. The format is `<PROJECT_KEY>-<NUMBER>` (e.g. `TF-141`, `PROJ-123`). The project key is everything before the last hyphen-number, and the number is the trailing integer.

## 2. Fetch Work Item Details

Use `mcp__taskwondo__get_work_item` with the display ID to get the full work item details (title, description, status, priority, type, labels).

Read and understand the work item requirements before proceeding.

## 3. Check Available Statuses

Use `mcp__taskwondo__list_statuses` with the project key to get the available workflow statuses. Identify the correct status names for "in progress" and "in review" categories — status names are case-sensitive and vary by project workflow.

## 4. Update Status to In Progress

Use `mcp__taskwondo__update_work_item` to set the work item status to the `in_progress` category status.

## 5. Plan the Implementation

Before writing code, review the codebase to understand:
- Which files need to be created or modified
- Existing patterns and conventions (see AGENTS.md)
- How similar features are implemented

Present a brief implementation plan and get user approval before proceeding.

## 6. Implement the Changes

Follow these conventions:
- **Go code**: Follow patterns in `api/internal/` — handler → service → repository dependency direction, zerolog logging, context as first param, interfaces defined by consumer
- **React/TypeScript code**: Follow patterns in `web/src/` — all UI strings in i18n, `useTranslation()` hook, destructive actions use `<Modal>`, path alias `@/` → `src/`
- **Commit messages**: Prefix with `[DISPLAY_ID]` (e.g. `[TF-141]`), where DISPLAY_ID is the actual work item display ID

Add progress comments to the work item as you complete significant milestones using `mcp__taskwondo__add_comment`. Keep comments concise and technical.

## 7. Validate with Tests

All changes MUST be validated with tests. Run both:

### Unit Tests
```bash
make test
```
This runs Go API tests (`cd api && go test ./... -v -race -cover`) and frontend tests (`cd web && npm test`).

For faster iteration during development, you can run targeted tests:
```bash
cd api && go test ./internal/handler/... -v -race       # One package
cd api && go test ./internal/service/... -v -run TestName # Single test
```

### E2E Tests
```bash
make test-e2e
```
This runs the full Playwright E2E suite in an isolated Docker stack.

**Important:**
- Write new unit tests for any new Go handlers, services, or repository methods
- Write or update E2E test scenarios for any user-facing behavior changes (tests live in `e2e/tests/` organized by domain)
- All tests must pass before considering the work complete
- If tests fail, fix the issues and re-run until green

## 8. Commit the Changes

Create a commit with the display ID prefix:
```
[DISPLAY_ID] Short description of the change
```

Do NOT push or create a pull request unless the user explicitly asks.

## 9. Post Implementation Summary

After all tests pass and changes are committed:

1. Get the commit SHA:
   ```bash
   git rev-parse HEAD
   ```

2. Construct the GitHub commit URL: `https://github.com/marcoshack/taskwondo/commit/<SHA>`

3. Add an implementation summary comment to the work item using `mcp__taskwondo__add_comment`:
   ```
   ## Implementation Summary

   <Brief description of what was implemented and how>

   ### Changes
   - <list of key files changed and why>

   ### Tests
   - <summary of tests added/updated>

   ### Commit
   <GitHub commit URL>
   ```

4. Update the work item status to the `in_review` category status using `mcp__taskwondo__update_work_item`.

## Checklist

Before marking complete, verify:
- [ ] Work item status set to In Progress at start
- [ ] Implementation follows project conventions (AGENTS.md)
- [ ] Unit tests written and passing (`make test`)
- [ ] E2E tests written/updated and passing (`make test-e2e`)
- [ ] Commit message prefixed with `[DISPLAY_ID]`
- [ ] Implementation summary comment added to work item
- [ ] Work item status set to In Review
