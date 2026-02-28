---
description: Run security scan and code quality review on changed files
argument-hint: Optional --skip-security or --skip-quality to skip a layer
---

# /sl-audit

Run a security vulnerability scan and code quality review on changed files. Both layers run independently and produce separate reports.

## Prerequisites

Follow the **state-management** skill for all state operations in this command.

## Step 1: Validate State

Read `.shiploop/state.json`.

- If `.shiploop/` doesn't exist: tell the user to run `/sl-status --init` first. Stop.
- Valid transition: `test → audit`
- If current phase is not `test`: tell the user the current phase and suggest the appropriate next command
- If current phase is already `audit`: this is a re-run — proceed

**If tests failed:** Read `.shiploop/audits/latest-tests.json`. If tests failed, warn:
"⚠ Tests had {N} failures. Proceeding with audit — security and quality issues are independent of test results."

## Step 2: Parse Input

Check `$ARGUMENTS` for flags:
- `--skip-security`: Skip the security scan (useful during rapid iteration)
- `--skip-quality`: Skip the quality review

Also read `.shiploop/config.yaml` for:
- `audit.skip_security`: default override
- `audit.skip_quality`: default override

Flags override config. If both security and quality are skipped: report "Both audit layers skipped. Run `/sl-gate` to proceed." Transition and stop.

## Step 3: Transition State

Transition to `audit` phase immediately (before starting scans):

1. Update `phase` to `audit`
2. Update `last_transition`
3. Update `updated_at`
4. Write state atomically

Log the transition to `decisions.jsonl`.

## Step 4: Run Security Audit

If security is not skipped:

Launch the **security-auditor** agent:
```
Scan these changed files for security vulnerabilities:

Changed files (from git diff):
{list of changed files}

Write findings to `.shiploop/audits/latest-security.json`.
Scope: only changed files and their direct imports.
```

## Step 5: Run Quality Review

If quality is not skipped:

Launch the **quality-reviewer** agent:
```
Review these changed files for code quality and convention adherence:

Changed files (from git diff):
{list of changed files}

Write findings to `.shiploop/audits/latest-quality.json`.
Read neighboring files to establish conventions before reviewing.
```

**Note:** Steps 4 and 5 can run in parallel — they have no dependencies on each other.

## Step 6: Aggregate Results

Read the reports that were generated:
- `.shiploop/audits/latest-security.json` (if security was run)
- `.shiploop/audits/latest-quality.json` (if quality was run)

Compute:
- **Blocking findings**: high-severity security issues
- **Advisory findings**: medium/low security + quality warnings
- **Informational**: info-level from both

## Step 7: Report

Show the user:

```
Audit complete

Security: {high} high, {medium} medium, {low} low, {info} info
Quality:  {errors} errors, {warnings} warnings, {info} info

{If blocking findings:}
⛔ Blocking:
  {severity} {rule}: {message} ({file}:{line})

{If advisory findings:}
⚠ Advisory:
  {severity} {category}: {message} ({file}:{line})

Reports:
  .shiploop/audits/latest-security.json
  .shiploop/audits/latest-quality.json

Next: Run /sl-gate for pre-merge review.
```

If security or quality was skipped, note it: "Security: skipped" or "Quality: skipped"