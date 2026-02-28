---
description: Execute the implementation plan using builder and boilerplate-gen agents
argument-hint: Optional --step N to run a single step
---

# /sl-build

Execute the current implementation plan step by step, routing each step to the appropriate agent based on its tag.

## Prerequisites

Follow the **state-management** skill for all state operations in this command.

## Step 1: Validate State

Read `.shiploop/state.json`.

- If `.shiploop/` doesn't exist: tell the user to run `/sl-status --init` first. Stop.
- Valid transition: `plan → implement`
- If current phase is not `plan`: tell the user the current phase and suggest the appropriate next command
- If current phase is already `implement`: this is a resume — continue from where we left off

## Step 2: Read Plan

Read `.shiploop/plans/current.md`.

If the file doesn't exist or is empty:
- Tell the user: "No plan found. Run `/sl-plan` first."
- Stop.

Parse the plan into numbered steps. For each step, extract:
- Step number and title
- Tag: `scaffold` or `implement`
- File paths
- Description

## Step 3: Transition State

Transition to `implement` phase immediately (before starting work):

1. Update `phase` to `implement`
2. Update `last_transition`
3. Update `updated_at`
4. Write state atomically

Log the transition to `decisions.jsonl`.

## Step 4: Execute Steps

### Check for --step flag

If `$ARGUMENTS` contains `--step N`, execute only step N. Otherwise, execute all steps in order.

### Create a TodoWrite list

Create a todo item for each step: "Step {N}: {title}"

### For each step, route by tag:

**If tag is `scaffold`:**

Launch the **boilerplate-gen** agent:
```
Execute this scaffold step:

Step {N}: {title}
Files: {file paths}
Description: {description}

Read existing files nearby to match conventions before creating new files.
```

**If tag is `implement`:**

Launch the **builder** agent:
```
Execute this implementation step:

Step {N}: {title}
Files: {file paths}
Description: {description}

Read the existing code first. Follow existing patterns. Do not add features beyond what the step describes.
```

### After each step:
- Mark the todo item as completed
- If the agent reported a failure, log it and continue to the next step (don't block the entire build)

### Parallelization

Steps MUST execute sequentially — later steps may depend on earlier ones. Do NOT run steps in parallel.

Exception: multiple consecutive `scaffold` steps with no shared file paths CAN run in parallel.

## Step 5: Verify Build

After all steps complete:

1. Check if the project has a build command (look for `package.json` scripts, `Makefile`, `Cargo.toml`, `pyproject.toml`, etc.)
2. If a build command exists, run it
3. Report build pass/fail

## Step 6: Transition to Test

Check step results before transitioning:

- If **all steps failed** (0 succeeded): do NOT advance. Report failures and stop. The user must fix the plan or re-run failed steps with `/sl-build --step N`.
- If **some steps failed** but at least one succeeded: warn the user and ask for confirmation before advancing to `test`. Show which steps failed.
- If **all steps succeeded**: advance automatically.

When advancing:

1. Transition state: `implement → test`
2. Log the transition with context about completed/failed steps
3. Write state atomically

## Step 7: Report

Show the user:

```
Build complete → {completed}/{total} steps executed
Phase: plan → implement → test

Steps:
  ✓ Step 1: {title} (scaffold)
  ✓ Step 2: {title} (implement)
  ✗ Step 3: {title} (implement) — {failure reason}
  ✓ Step 4: {title} (scaffold)

Build: {PASS | FAIL | no build command found}

Next: Run /sl-test to generate and run tests.
```

If any steps failed, also suggest: "Re-run failed steps with `/sl-build --step N`"