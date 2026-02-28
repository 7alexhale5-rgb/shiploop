---
description: Break the current spec into a numbered implementation plan with file targets
argument-hint: Optional flags (none currently)
---

# /sl-plan

Read the current spec and generate a step-by-step implementation plan.

## Prerequisites

Follow the **state-management** skill for all state operations in this command.

## Step 1: Validate State

Read `.shiploop/state.json`.

- If `.shiploop/` doesn't exist: tell the user to run `/sl-status --init` first. Stop.
- Valid transition: `spec → plan`
- If current phase is not `spec`: tell the user the current phase and suggest the appropriate next command
- If current phase is already `plan`: warn "Overwriting existing plan" and continue

## Step 2: Read Spec

Read `.shiploop/specs/current.md`.

If the file doesn't exist or is empty:
- Tell the user: "No spec found. Run `/sl-spec` first."
- Stop.

## Step 3: Generate Plan

Launch the **planner** agent with this context:

```
Create an implementation plan for this spec:

---
{contents of specs/current.md}
---

Analyze the codebase thoroughly before writing the plan.
Each step must have a tag: `scaffold` or `implement`.
Each step must reference specific file paths.
Follow the plan format exactly as defined in your instructions.
```

## Step 4: Write Plan

Take the planner's output and write it to `.shiploop/plans/current.md`.

If `plans/` directory doesn't exist inside `.shiploop/`, create it.

## Step 5: Transition State

Follow the state-management skill's **Transition Protocol**:

1. Read current state
2. Update `phase` to `plan`
3. Set `plan_ref` to `plans/current.md`
4. Update `last_transition` with `from` (`spec`) and `to` (`plan`)
5. Update `updated_at`
6. Write state atomically

## Step 6: Log Decision

Append to `.shiploop/log/decisions.jsonl`:

```jsonl
{"ts":"<ISO-8601>","phase":"plan","event":"transition","from":"spec","to":"plan","actor":"system","context":{"plan_ref":"plans/current.md","step_count":<N>,"scaffold_steps":<N>,"implement_steps":<N>}}
```

## Step 7: Report

Show the user:

```
Plan written → .shiploop/plans/current.md
Phase: spec → plan
Steps: {total} ({implement_count} implement + {scaffold_count} scaffold)

Next: Run /sl-build to start implementation.
```

Then display the step titles as a numbered list so the user can review before building.