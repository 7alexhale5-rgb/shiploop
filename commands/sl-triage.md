---
description: Triage detected issue — confirm, analyze root cause, decide to fix or dismiss
argument-hint: Optional --dismiss (skip analysis, dismiss as noise), --force-reentry (skip confirmation)
---

# /sl-triage

Multi-gate command that handles the issue triage flow. Covers two decision points depending on current state: confirming an issue exists (gate_issue_detected) and approving re-entry into the loop (gate_re_entry).

## Prerequisites

Follow the **state-management** skill for all state operations in this command.

## Step 1: Validate State

Read `.shiploop/state.json`.

- If `.shiploop/` doesn't exist: tell the user to run `/sl-status --init` first. Stop.
- Valid entry states: `gate_issue_detected`, `triage`, `gate_re_entry`
- If current phase is `gate_issue_detected`: proceed to **Gate A** (confirm/dismiss)
- If current phase is `triage`: proceed to **Step 4** (analysis already approved, launch agent)
- If current phase is `gate_re_entry`: proceed to **Gate B** (approve re-entry/cancel)
- If current phase is anything else: tell the user the current phase and suggest the appropriate next command. Stop.

## Step 2: Parse Input

Check `$ARGUMENTS` for flags:
- `--dismiss`: Immediately dismiss the issue as noise (shortcut — skips Gate A confirmation)
- `--force-reentry`: Skip Gate B confirmation and immediately approve re-entry

## Step 3: Gate A — Confirm or Dismiss Issue

**Entry state: `gate_issue_detected`**

Read `.shiploop/observations/latest-issue.json` for issue details.

Present the issue summary:
```
Issue detected during observation:

Source: {Sentry | CI logs | manual}
Deploy: {tag} ({commit})

Issues:
  ⚠ {title} ({count} occurrences)
  ⚠ {title} ({count} occurrences)
```

**If `--dismiss` was passed:** skip to dismiss path below.

Ask the user to choose:
- **Confirm** → this is a real issue, proceed to triage analysis
- **Dismiss** → this is noise (false positive, transient, pre-existing)

**If Confirm:**
1. Transition state: `gate_issue_detected → triage`
2. Log to `decisions.jsonl`: `{event: "gate_decision", phase: "gate_issue_detected", decision: "confirm", actor: "human", issues: [...]}`
3. Proceed to Step 4

**If Dismiss:**
1. Transition state: `gate_issue_detected → observe`
2. Log to `decisions.jsonl`: `{event: "gate_decision", phase: "gate_issue_detected", decision: "dismiss", actor: "human", reason: "{user's reason or 'noise'}"}`
3. Report:
```
Issue dismissed → returning to observation

Phase: gate_issue_detected → observe
Reason: {reason}

Next: Run /sl-observe to continue monitoring, or /sl-observe --done if satisfied.
```
4. Stop.

## Step 4: Launch Triage Analyst

**Entry state: `triage`**

Read `.shiploop/config.yaml` for triage settings:
- `triage.max_iterations` (default: 5)
- `triage.archive_specs` (default: true)

Read current iteration from `.shiploop/state.json`.

**Check iteration cap:**
If `iteration >= max_iterations`:
```
⚠ Iteration cap reached ({iteration}/{max_iterations})

This issue has persisted across {iteration} fix attempts.
Recommend: manual investigation instead of another automated loop.

Override: set triage.max_iterations higher in .shiploop/config.yaml
```
Ask user: **Continue anyway** or **Stop** (transition to idle).

Launch the **triage-analyst** agent:
```
Analyze the production issue and draft a fix spec:

Issue data: .shiploop/observations/latest-issue.json
Original spec: .shiploop/specs/current.md
Deploy ref: {tag} ({commit})
Iteration: {N}
```

## Step 5: Archive + Promote Spec

After the triage-analyst completes:

**If `archive_specs: true`:**
1. Copy `.shiploop/specs/current.md` to `.shiploop/specs/iteration-{N}.md`
   - Use: `cp .shiploop/specs/current.md .shiploop/specs/iteration-{N}.md`
2. Copy `.shiploop/specs/fix-draft.md` to `.shiploop/specs/current.md`
   - Use: `cp .shiploop/specs/fix-draft.md .shiploop/specs/current.md`

**If `archive_specs: false`:**
1. Overwrite `.shiploop/specs/current.md` with `.shiploop/specs/fix-draft.md`
   - Use: `cp .shiploop/specs/fix-draft.md .shiploop/specs/current.md`

Transition state: `triage → gate_re_entry`
Log to `decisions.jsonl`: `{event: "transition", phase: "triage", to: "gate_re_entry", fix_spec: "specs/current.md", archived_as: "specs/iteration-{N}.md"}`

Report the triage-analyst's findings (from agent output).

## Step 6: Gate B — Approve Re-entry or Cancel

**Entry state: `gate_re_entry`**

Read the fix spec from `.shiploop/specs/current.md`.
Read current iteration from `.shiploop/state.json`.

Present the fix spec summary:
```
Fix spec ready for re-entry:

Iteration: {N} → {N+1}
Issue: {title}
Classification: {type}
Fix complexity: {level}

Proposed fix: {1-2 sentence summary from spec}
Files affected: {N}
```

**If `--force-reentry` was passed:** skip to approve path below.

Ask the user to choose:
- **Approve re-entry** → loop back to spec with the fix spec, start a new iteration
- **Cancel** → abandon the loop, return to idle

**If Approve:**
1. Increment `iteration` in state
2. Transition state: `gate_re_entry → spec`
3. Log to `decisions.jsonl`: `{event: "gate_decision", phase: "gate_re_entry", decision: "approve", actor: "human", iteration: {N+1}}`
4. Report:
```
Re-entry approved → looping back to spec

Phase: gate_re_entry → spec (iteration {N+1})
Fix spec: .shiploop/specs/current.md
Archived previous: .shiploop/specs/iteration-{N}.md

Next: The fix spec is loaded. Run /sl-plan to generate an implementation plan.
  Or run /sl-loop to auto-advance through the pipeline.
```

**If Cancel:**
1. Transition state: `gate_re_entry → idle`
2. Log to `decisions.jsonl`: `{event: "gate_decision", phase: "gate_re_entry", decision: "cancel", actor: "human", reason: "{user's reason}"}`
3. Report:
```
Loop cancelled → returning to idle

Phase: gate_re_entry → idle
Iteration {N} ended without fix.
Reason: {reason}

The issue remains open. Address it manually or start fresh with /sl-spec.
```
