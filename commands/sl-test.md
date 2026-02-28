---
description: Generate and run targeted unit tests for changed code (JiTTest)
argument-hint: Optional --keep to preserve test artifacts, --skip-run to generate only
---

# /sl-test

Generate targeted unit tests for code that changed since the last spec, run them, and report results.

## Prerequisites

Follow the **state-management** skill for all state operations in this command.

## Step 1: Validate State

Read `.shiploop/state.json`.

- If `.shiploop/` doesn't exist: tell the user to run `/sl-status --init` first. Stop.
- Valid transition: `implement → test`
- If current phase is not `implement`: tell the user the current phase and suggest the appropriate next command
- If current phase is already `test`: this is a re-run — proceed (regenerate tests and re-execute)

## Step 2: Parse Input

Check `$ARGUMENTS` for flags:
- `--keep`: Preserve generated test files in `.shiploop/tests/` after execution (default: delete after run)
- `--skip-run`: Generate test files only, do not execute them. Useful for manual review before running.

## Step 3: Detect Changes

1. Run `git diff --name-only` to identify changed files (no `HEAD` — detects uncommitted working-tree changes from the build step)
2. Filter to source files only — exclude:
   - Config files (`.json`, `.yaml`, `.toml`, `.env`)
   - Documentation (`.md`, `.txt`)
   - Generated files (`*.min.*`, `dist/`, `build/`)
   - Test files (`*.test.*`, `*.spec.*`)
3. Read the spec from `.shiploop/specs/current.md` for intent context
4. If no source files changed: report "No source changes detected — nothing to test." Transition to `test` and stop.

## Step 4: Generate Tests

Create the directory `.shiploop/tests/` if it doesn't exist.

Launch the **test-generator** agent:
```
Generate targeted unit tests for these changed files:

Changed files:
{list of changed source files}

Spec context:
{contents of .shiploop/specs/current.md}

Write test files to `.shiploop/tests/`. Match the project's test framework.
```

## Step 5: Run Tests

If `--skip-run` was passed, skip this step. Report "Tests generated but not executed (--skip-run)."

Otherwise:

1. **Detect test runner:**
   - Look for `jest.config.*` or `"jest"` in `package.json` → run `npx jest .shiploop/tests/`
   - Look for `vitest.config.*` or `"vitest"` in `package.json` → run `npx vitest run .shiploop/tests/`
   - Look for `pytest.ini`, `pyproject.toml`, or `conftest.py` → run `python -m pytest .shiploop/tests/`
   - Look for `*_test.go` → run `go test ./.shiploop/tests/...`
   - If none found: try `npx vitest run .shiploop/tests/` as default for JS/TS projects
   - Read `.shiploop/config.yaml` `test.runner` for explicit override

2. **Execute with timeout** — Use the timeout from config (default 120s). Capture stdout/stderr.

3. **Parse results** — Extract pass/fail counts and any coverage output.

## Step 6: Write Results

Write `.shiploop/audits/latest-tests.json`:
```json
{
  "generated_at": "ISO-8601 timestamp",
  "test_files": ["list of generated test files"],
  "total": 12,
  "passed": 11,
  "failed": 1,
  "failures": [{"file": "tests/foo.test.ts", "test": "should validate input", "error": "Expected true, got false"}],
  "coverage": {"lines": 0.82, "branches": 0.71},
  "spec_ref": "specs/current.md",
  "diff_summary": "3 files changed, 45 insertions, 12 deletions"
}
```

If `--skip-run`, write results with `"total": 0, "passed": 0, "failed": 0` and note `"status": "generated_only"`.

**Cleanup:** Unless `--keep` was passed, delete all files in `.shiploop/tests/` after writing results.

## Step 7: Transition + Report

1. Transition state: `implement → test` (or stay at `test` if re-running)
2. Log to `decisions.jsonl`: phase transition, test counts, pass/fail
3. Report to user:

```
JiTTest complete → {passed}/{total} tests passed

Generated: {N} test files for {M} changed source files
Runner: {vitest|jest|pytest|go test}
Results: {passed} passed, {failed} failed
Coverage: {lines}% lines, {branches}% branches

{If failures exist:}
Failures:
  ✗ {test name} — {error summary}

Next: Run /sl-audit to scan for security and quality issues.
```

If `--skip-run`: "Tests generated in `.shiploop/tests/`. Review them, then re-run `/sl-test` without `--skip-run`."