---
description: Master orchestrator — read state and chain to the next lifecycle command
argument-hint: Optional feature description to pass to /sl-spec when starting from idle. --fast for trivial changes. --force to bypass iteration cap.
---

# /sl-loop

The "loop" in ShipLoop. Reads the current state and chains to the next command in the lifecycle. Run this to auto-advance through the pipeline without remembering which command comes next.

## Prerequisites

Follow the **state-management** skill for all state operations in this command.

## Step 1: Validate State

Check if `.shiploop/` exists.

**If `.shiploop/` doesn't exist:**
1. Initialize the project — create the full `.shiploop/` directory structure per the state-management skill
2. Set state to `idle`
3. Report: "ShipLoop initialized. State → idle."

Read `.shiploop/state.json` for current phase and iteration.

## Step 2: Read Config

Read `.shiploop/config.yaml` for:
- `triage.max_iterations` (default: 5)
- `gates.*.auto_approve` settings

## Step 3: Fast Lane Detection

The fast lane collapses spec + plan + build into a single inline step for trivial changes. Audit and gate still run — governance is never skipped.

**Trigger conditions (ALL must be true):**
- `--fast` flag is passed in `$ARGUMENTS`, OR auto-detected (see below)
- Current phase is `idle` (starting fresh)
- A feature description is provided in `$ARGUMENTS`

**Auto-detection (when `--fast` is NOT explicit):**
Skip auto-detection. The fast lane only activates with `--fast`. This prevents surprising behavior on changes that look small but have hidden complexity.

**When fast lane activates:**

1. Report: "Fast lane: trivial change detected — collapsing spec + plan + build."
2. **Inline spec**: Write a minimal spec to `.shiploop/specs/current.md` directly (no spec-writer agent):
   ```markdown
   # Quick Spec
   **Description:** {user's feature description}
   **Requirements:**
   - [ ] {single requirement derived from description}
   **Scope:** Quick fix — single file, <10 LOC
   ```
3. Transition state: `idle → spec`
4. **Inline plan**: Analyze the codebase yourself to identify the target file and change. Write a 1-step plan to `.shiploop/plans/current.md`:
   ```markdown
   # Implementation Plan
   **Spec:** {description}
   **Estimated steps:** 1
   ## Steps
   ### 1. {action}
   - **Tag:** `implement`
   - **Files:** `{file}` (modify)
   - **Description:** {what to change}
   ```
5. Transition state: `spec → plan`
6. **Inline build**: Make the code change directly — no builder agent. Read the file, make the edit, verify syntax.
7. Transition state: `plan → implement`
8. Log all transitions to `decisions.jsonl` with `"mode": "fast_lane"`
9. Report what was done, then continue to `/sl-test` as normal (the audit + gate phases still run in full)

**Fast lane does NOT skip:** test, audit, gate, ship, observe. Only spec/plan/build are collapsed.

## Step 4: Check Iteration Cap

If `iteration >= triage.max_iterations` AND `--force` was NOT passed:
```
⚠ Iteration cap reached ({iteration}/{max_iterations})

{iteration} fix iterations have run without resolving the issue.
Recommend: manual investigation or increase triage.max_iterations in config.

Override: Run /sl-loop --force to bypass the cap for one cycle.
Or run /sl-triage manually to continue, or /sl-status to review history.
```
Stop — do not auto-advance past the iteration cap.

If `--force` was passed: log `{event: "iteration_cap_override", iteration: N, actor: "human"}` to `decisions.jsonl` and continue past the cap for this invocation only.

## Step 5: Route to Next Command

Based on the current phase, determine and invoke the next command:

| Current Phase | Next Action | Command |
|---------------|-------------|---------|
| `idle` | Start new feature | `/sl-spec {$ARGUMENTS if provided}` |
| `spec` | Generate plan from spec | `/sl-plan` |
| `plan` | Execute the plan | `/sl-build` |
| `implement` | Run tests | `/sl-test` |
| `test` | Run audits | `/sl-audit` |
| `audit` | Present gate | `/sl-gate` |
| `gate_pre_merge` | **Waiting for human decision** | _(see below)_ |
| `ship` | Start observation | `/sl-observe` |
| `observe` | **Waiting for result** | _(see below)_ |
| `gate_issue_detected` | Triage the issue | `/sl-triage` |
| `triage` | **Analysis in progress** | _(see below)_ |
| `gate_re_entry` | **Waiting for human decision** | _(see below)_ |

### Human Decision States

For phases that require human input, do NOT auto-advance. Instead, report the state and prompt:

**`gate_pre_merge`:**
```
Waiting for gate decision.

The pre-merge gate is ready for review.
Run /sl-gate to see the summary and approve or reject.
```

**`observe`:**
```
Observation in progress.

The deploy is being monitored.
Run /sl-observe --done (clean) or /sl-observe --issue "desc" (problem found).
```

**`triage`:**
```
Triage analysis in progress.

Run /sl-triage to continue the triage flow.
```

**`gate_re_entry`:**
```
Waiting for re-entry decision.

A fix spec has been drafted. Review and decide:
Run /sl-triage to approve re-entry or cancel.
```

### Auto-advance

For non-gate phases, report what's happening and invoke the next command:
```
ShipLoop: {current_phase} → advancing to {next_phase}
Iteration: {N}

Running /{next_command}...
```

Then invoke the command. When that command completes, do NOT automatically chain to the next — return control to the user. The user runs `/sl-loop` again to advance further.

**Exception — idle with arguments:** If the current phase is `idle` and `$ARGUMENTS` contains a feature description, pass it directly to `/sl-spec`:
```
ShipLoop: Starting new feature
Description: {arguments}

Running /sl-spec...
```

## Step 6: Report

After the chained command completes, report the new state:
```
ShipLoop status:
  Phase: {new_phase}
  Iteration: {N}
  Last action: {what just happened}

Next: Run /sl-loop to continue, or /sl-status for full details.
```
