---
description: Generate or refine a spec from requirements, a file, or a production observation
argument-hint: "requirements text" | path/to/file | --from-observation
---

# /sl-spec

Generate a structured implementation spec and write it to `.shiploop/specs/current.md`.

## Prerequisites

Follow the **state-management** skill for all state operations in this command.

## Step 1: Validate State

Read `.shiploop/state.json`.

- If `.shiploop/` doesn't exist: tell the user to run `/sl-status --init` first. Stop.
- Valid transitions into `spec`: from `idle`, from `gate_re_entry` (loop restart)
- If current phase is already `spec`: warn "Overwriting existing spec" and continue
- If current phase is any other active phase (plan, implement, test, audit, etc.): warn that this will restart the loop and ask for confirmation before proceeding

## Step 2: Parse Input

The user's input is in `$ARGUMENTS`. Determine the input type:

### Inline text (default)
If `$ARGUMENTS` does not start with `/` or `--`, treat it as raw requirements text.

### File path
If `$ARGUMENTS` starts with `/` or contains a file extension (`.md`, `.txt`, etc.), read the file at that path.

### From observation
If `$ARGUMENTS` contains `--from-observation`:
- Read `.shiploop/observations/latest-issue.json`
- If it doesn't exist, tell the user: "No observations found. Run `/sl-observe` first, or provide requirements directly."
- Extract the error context, affected files, and severity to use as requirements input

### No arguments
If `$ARGUMENTS` is empty, ask the user what they want to build. Do not proceed without input.

## Step 3: Generate Spec

Launch the **spec-writer** agent with this context:

```
Write a spec for the following requirements:

{parsed input from Step 2}

The spec will be written to .shiploop/specs/current.md.
Follow the spec format exactly as defined in your instructions.
```

## Step 4: Write Spec

Take the spec-writer's output and write it to `.shiploop/specs/current.md`.

If `specs/` directory doesn't exist inside `.shiploop/`, create it.

## Step 5: Transition State

Follow the state-management skill's **Transition Protocol**:

1. Read current state
2. Update `phase` to `spec`
3. Set `spec_ref` to `specs/current.md`
4. Update `last_transition` with `from` (previous phase) and `to` (`spec`)
5. Update `updated_at` to current ISO-8601 timestamp
6. If this is the first transition from `idle`, set `started_at`
7. If transitioning from `gate_re_entry`, increment `iteration`
8. Write state atomically (temp file + rename)

## Step 6: Log Decision

Append to `.shiploop/log/decisions.jsonl`:

```jsonl
{"ts":"<ISO-8601>","phase":"spec","event":"transition","from":"<previous_phase>","to":"spec","actor":"system","context":{"source":"<inline|file|observation>","spec_ref":"specs/current.md"}}
```

## Step 7: Report

Show the user:

```
Spec written → .shiploop/specs/current.md
Phase: idle → spec (iteration {n})

Next: Run /sl-plan to generate an implementation plan.
```

Then display a brief summary of the spec (title + requirements count + scope).