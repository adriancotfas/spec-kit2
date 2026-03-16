---
description: Validate the full implementation end-to-end against every user story in the spec, run all existing test suites, and produce a list of any remaining manual verification tasks.
scripts:
  sh: scripts/bash/check-prerequisites.sh --json --require-tasks --include-tasks
  ps: scripts/powershell/check-prerequisites.ps1 -Json -RequireTasks -IncludeTasks
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Pre-Execution Checks

**Check for extension hooks (before verification)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.before_verify` key
- If the YAML cannot be parsed or is invalid, skip hook checking silently and continue normally
- Filter to only hooks where `enabled: true`
- For each remaining hook, do **not** attempt to interpret or evaluate hook `condition` expressions:
  - If the hook has no `condition` field, or it is null/empty, treat the hook as executable
  - If the hook defines a non-empty `condition`, skip the hook and leave condition evaluation to the HookExecutor implementation
- For each executable hook, output the following based on its `optional` flag:
  - **Optional hook** (`optional: true`):
    ```
    ## Extension Hooks

    **Optional Pre-Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
    ```
  - **Mandatory hook** (`optional: false`):
    ```
    ## Extension Hooks

    **Automatic Pre-Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}

    Wait for the result of the hook command before proceeding to the Outline.
    ```
- If no hooks are registered or `.specify/extensions.yml` does not exist, skip silently

## Outline

### 1. Initialise Context

Run `{SCRIPT}` from repo root and parse `FEATURE_DIR` and `AVAILABLE_DOCS`. All paths must be absolute.
For single quotes in args like "I'm Groot", use escape syntax: e.g `'I'\''m Groot'` (or double-quote if possible: `"I'm Groot"`).

Load the following documents from `FEATURE_DIR`:

- **REQUIRED**: `spec.md` — source of truth for user stories and acceptance criteria
- **REQUIRED**: `plan.md` — tech stack, architecture, project structure
- **IF EXISTS**: `tasks.md` — task breakdown (used to cross-reference implemented scope)
- **IF EXISTS**: `data-model.md` — entities and relationships
- **IF EXISTS**: `contracts/` — API contracts and interface definitions
- **IF EXISTS**: `quickstart.md` — integration and smoke-test scenarios

### 2. Build the Verification Checklist

Parse `spec.md` and extract **every user story** (including edge cases and non-functional requirements).
For each user story produce one or more verification items.

Output a structured checklist in this format:

```
## Verification Checklist

### US-01: <user story title>
- [ ] <concrete, observable check derived from acceptance criteria>
- [ ] <additional check if multiple criteria exist>

### US-02: <user story title>
- [ ] <check>
...
```

Assign one of the following verification **strategies** to each item (you will use this in step 4):

| Strategy | Description |
|---|---|
| `auto:unit` | Covered by an existing unit / integration test |
| `auto:api` | Verifiable by calling a local HTTP endpoint |
| `auto:ui` | Verifiable by rendering and inspecting a browser screen (Playwright MCP) |
| `auto:cli` | Verifiable by executing a CLI command and inspecting output |
| `auto:file` | Verifiable by inspecting generated or transformed files |
| `manual` | Cannot be automated — requires a human |

Items that can be automated in any `auto:*` way **must not** be assigned `manual`.  
Only use `manual` for checks that are genuinely impossible without human involvement (e.g. physical hardware, third-party payment flows with real credentials, live production services, subjective aesthetic judgements).

### 3. Detect Local Runtime Environment

**Priority order — stop at the first successful match:**

1. **Project config files** (check these first, in order):
   - `package.json` → look for `scripts.start`, `scripts.dev`, `scripts.serve`; identify start command and port
   - `docker-compose.yml` / `docker-compose.yaml` / `compose.yml` — identify service names, ports, and start command
   - `Procfile` — identify web/worker processes and commands
   - `Makefile` — look for `run`, `start`, `dev`, `serve` targets
   - `.env` / `.env.local` / `.env.development` — extract `PORT`, `HOST`, `BASE_URL`, `API_URL`
   - `pyproject.toml` / `setup.cfg` — look for `[tool.pytest.ini_options]` test command; identify framework (FastAPI, Flask, Django)
   - `Cargo.toml` — detect Rust binary target and test command
   - `go.mod` — detect Go module; identify main package
   - `pom.xml` / `build.gradle` — detect Java/Kotlin application with embedded server (Spring Boot, Quarkus)
   - `.ruby-version` / `Gemfile` — detect Rails / Sinatra and rake/rails server commands
   - `angular.json` / `next.config.*` / `vite.config.*` / `nuxt.config.*` — detect frontend framework and dev-server port
   - `.rspec` / `jest.config.*` / `vitest.config.*` / `pytest.ini` / `phpunit.xml` — identify existing test runners

2. **plan.md tech stack** — if no config files matched, parse the architecture section for the runtime, framework, and typical port.

3. **Source code analysis** — as a last resort, inspect `main.*`, `app.*`, `server.*`, `index.*` in the project root or `src/` for framework bootstrapping patterns.

Produce a concise **Environment Summary**:

```
## Environment Summary

Runtime:       <e.g. Node.js 20 / Python 3.12 / Go 1.22>
Framework:     <e.g. Next.js / FastAPI / Gin>
Start command: <e.g. npm run dev / uvicorn app.main:app --reload / go run ./cmd/server>
Base URL:      <e.g. http://localhost:3000>
Test runner:   <e.g. jest / pytest / go test ./...>
Detection:     <which config file or strategy was used>
```

If no local environment can be determined (e.g. the project is a library with no server), note it and proceed — all API and UI checks will be classified `manual` automatically.

### 4. Run Existing Test Suites

**Before any E2E verification**, execute the existing automated test suite(s) detected in step 3.

- Run the test command(s) and capture output.
- Report a **Test Suite Summary** table:

  ```
  | Suite         | Result | Tests | Passed | Failed | Skipped |
  |---------------|--------|-------|--------|--------|---------|
  | jest          | ✓ PASS | 42    | 42     | 0      | 0       |
  | cypress e2e   | ✗ FAIL | 10    | 8      | 2      | 0       |
  ```

- **Failing tests do not stop execution** — continue to E2E verification but note all failures in the final report.
- If no test runner is configured, note this and continue.

> **Important**: Existing test suites are complementary, not a substitute for E2E verification. Unit tests may have gaps; the E2E steps below cover every user story regardless.

### 5. E2E Verification Pass

Work through **every item** in the checklist built in step 2. For each item:

#### `auto:unit` items
Cross-reference the test suite output from step 4.  
Mark `✓ PASS`, `✗ FAIL`, or `⚠ NOT COVERED` based on whether a matching test exists and passes.

#### `auto:api` items
1. Ensure the local server is running (start it if needed using the detected start command; wait up to 30 s).
2. Construct a realistic HTTP request (method, path, headers, body) matching the spec.
3. Execute the request using an available tool (MCP HTTP client, `curl`, `fetch`, etc.).
4. Validate: status code, response body shape, and any spec-defined invariants.
5. Record the request + response summary alongside the result.

#### `auto:ui` items
1. Use **Playwright MCP** (if available) to:
   - Navigate to the relevant URL / route.
   - Interact with the UI as a real user would (click, type, submit).
   - Assert that the screen contains the expected elements, text, or state described in the acceptance criteria.
2. Capture a screenshot as evidence where helpful.
3. If Playwright MCP is not available, record the item as `manual`.

#### `auto:cli` items
1. Run the CLI command with representative inputs.
2. Assert the exit code and stdout/stderr match what the spec describes.

#### `auto:file` items
1. Trigger the relevant operation (e.g. export, generate, transform).
2. Inspect the resulting file(s) for expected content, format, or structure.

#### `manual` items
Do not attempt to execute; accumulate in the manual-tasks list (see step 7).

### 6. Produce the Results Table

After completing all automated checks, produce a summary table:

```
## Verification Results

| # | User Story | Check | Strategy | Result | Notes |
|---|-----------|-------|----------|--------|-------|
| 1 | US-01 Login | User can submit credentials | auto:api | ✓ PASS | POST /auth/login → 200 |
| 2 | US-01 Login | Error shown for bad password | auto:ui | ✓ PASS | Screenshot captured |
| 3 | US-02 Dashboard | Stats widget loads correctly | auto:ui | ✗ FAIL | Element not found: [data-testid="stats"] |
| 4 | US-03 Export | CSV file generated | auto:file | ✓ PASS | |
| 5 | US-04 Payment | Stripe checkout completes | manual | — | Real card needed |
```

Calculate an **overall status**:

- ✅ **COMPLETE** — all automated checks pass and no failures (manual items excluded)
- ⚠ **PARTIAL** — some automated checks fail or some user stories have no automated coverage
- ❌ **INCOMPLETE** — significant gaps or multiple failures

### 7. Manual Tasks List

Produce a dedicated section listing every item that requires human action:

```
## Manual Verification Tasks

The following items could not be verified automatically.
Please complete these checks manually before considering the implementation done.

1. **US-04 Stripe Payment** — Complete a real checkout using a physical test card (Stripe test card: 4242 4242 4242 4242). Verify confirmation email is received.

2. **US-07 SSO Login** — Sign in through the corporate SSO provider in a real browser session. Confirm the JWT returned is valid and contains expected claims.

3. **US-09 PDF Export** — Open the exported PDF in a PDF viewer and visually confirm layout, pagination, and font rendering match the design mockup.
```

If there are no manual tasks, state: _"No manual verification tasks — all user stories were verified automatically."_

### 8. Final Report

Produce a single concise final report:

```
## Verification Report

Feature:  <feature name from plan.md>
Date:     <today's date>
Overall:  ✅ COMPLETE / ⚠ PARTIAL / ❌ INCOMPLETE

### Summary
- Automated checks:  X pass / Y fail / Z not covered
- Manual tasks:      N remaining
- Test suites:       X pass / Y fail

### Failing Checks
(list any `✗ FAIL` items with remediation hints)

### Recommended Next Steps
(if PARTIAL or INCOMPLETE: specific fixes, re-runs, or implementation gaps to address)
```

---

**Check for extension hooks (after verification)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.after_verify` key
- If the YAML cannot be parsed or is invalid, skip hook checking silently and continue normally
- Filter to only hooks where `enabled: true`
- For each remaining hook, do **not** attempt to interpret or evaluate hook `condition` expressions:
  - If the hook has no `condition` field, or it is null/empty, treat the hook as executable
  - If the hook defines a non-empty `condition`, skip the hook and leave condition evaluation to the HookExecutor implementation
- For each executable hook, output the following based on its `optional` flag:
  - **Optional hook** (`optional: true`):
    ```
    ## Extension Hooks

    **Optional Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
    ```
  - **Mandatory hook** (`optional: false`):
    ```
    ## Extension Hooks

    **Automatic Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}
    ```
- If no hooks are registered or `.specify/extensions.yml` does not exist, skip silently
