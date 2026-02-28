---
description: Monitor post-deploy for issues via Sentry or manual observation
argument-hint: Optional --done (mark clean), --issue "desc" (report issue), --window 60 (override minutes)
---

# /sl-observe

Monitor the deployed release for issues. Uses Sentry MCP if available, falls back to manual observation. The observation ends with a human decision: clean (loop complete) or issue detected (proceed to triage).

## Prerequisites

Follow the **state-management** skill for all state operations in this command.

## Step 1: Validate State

Read `.shiploop/state.json`.

- If `.shiploop/` doesn't exist: tell the user to run `/sl-status --init` first. Stop.
- Valid transition: `ship → observe`
- If current phase is already `observe`: this is a re-check — proceed
- If current phase is not `ship` or `observe`: tell the user the current phase and suggest the appropriate next command

## Step 2: Parse Input

Check `$ARGUMENTS` for flags:
- `--done`: Immediately mark observation as clean (shortcut for manual observation)
- `--issue "description"`: Immediately report a manually observed issue
- `--window N`: Override observation window duration in minutes (default from config: 30)

If `--done` is passed, skip to Step 6 with result = CLEAN.
If `--issue` is passed, skip to Step 5 with the provided issue description.

## Step 3: Read Context

Read `.shiploop/audits/latest-ship.json` for deploy reference:
- Tag name, commit SHA, PR details
- This is what was deployed — errors before this deploy are not our concern

Read `.shiploop/config.yaml` for observe settings:
- `observe.window` (default: 30)
- `observe.sentry` (auto-detect if not set)

## Step 4: Launch Observer

Detect Sentry MCP availability — check if Sentry MCP tools are accessible in the current runtime.

Launch the **observer** agent:
```
Monitor post-deploy health:

Deploy ref: {tag} ({commit})
Deploy time: {ship timestamp from latest-ship.json}
Observation window: {window} minutes
Sentry available: {true|false}
```

Transition state to `observe` if not already there.

## Step 5: Process Results

Read the observer's findings from `.shiploop/observations/latest-issue.json`.

**If result is CLEAN:**
- No new errors detected
- CI is passing (or unavailable)
- Proceed to Step 6 with result = CLEAN

**If result is ISSUE:**
- New errors or CI failures detected
- Write issue details to `.shiploop/observations/latest-issue.json` (observer already did this)
- Proceed to Step 6 with result = ISSUE

**If manual observation (no Sentry, no --done/--issue):**
- Present the observation prompt to the user:
```
Observation window open ({window} minutes)
Deploy: {tag} ({commit})

Monitor your application for issues. When ready:
  /sl-observe --done          → mark as clean (no issues)
  /sl-observe --issue "desc"  → report an issue

Or wait — I'll check CI status periodically if gh is available.
```
- Stop here and wait for the user to re-invoke with `--done` or `--issue`

## Step 6: Human Decision + Transition

**If CLEAN:**
1. Transition state: `observe → idle` (loop complete!)
2. Log to `decisions.jsonl`: `{result: "clean", deploy_ref: "{tag}", observation_window: "{window}m"}`
3. Report:
```
Observation complete → CLEAN

Deploy: {tag} ({commit})
Window: {window} minutes
Source: {Sentry | CI logs | manual}
Result: No issues detected

✓ ShipLoop cycle complete! The feature has been shipped and verified.
  Iteration {N} finished. State → idle.

Next: Start a new feature with /sl-spec or run /sl-status to review.
```

**If ISSUE:**
1. Transition state: `observe → gate_issue_detected`
2. Log to `decisions.jsonl`: `{result: "issue", issues: [...], deploy_ref: "{tag}"}`
3. Report:
```
Observation complete → ISSUE DETECTED

Deploy: {tag} ({commit})
Window: {window} minutes
Source: {Sentry | CI logs | manual}

Issues found:
  ⚠ {issue title} ({count} occurrences)
  ⚠ {issue title} ({count} occurrences)

Next: Run /sl-triage to analyze and decide how to handle the issue.
  - If this is noise (not a real issue), /sl-triage will let you dismiss it.
  - If this is real, /sl-triage will draft a fix spec and loop back.
```
