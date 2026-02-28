---
description: Show current ShipLoop state and phase
argument-hint: Optional --init to initialize .shiploop/
---

# /sl-status

Show the current state of the ShipLoop lifecycle in this project.

## Behavior

### If `.shiploop/` does not exist

If the user passed `--init` as an argument ($ARGUMENTS contains "--init"):
- Initialize `.shiploop/` following the state-management skill
- Confirm initialization

Otherwise:
- Report that ShipLoop is not initialized in this project
- Suggest: "Run `/sl-status --init` to set up ShipLoop in this project."

### If `.shiploop/` exists

1. Read `.shiploop/state.json`
2. Read `.shiploop/config.yaml` (if exists)
3. Count entries in `.shiploop/log/decisions.jsonl` (if exists)
4. List handoffs in `.shiploop/handoffs/` (if any)

### Output Format

Display a concise status report:

```
ShipLoop Status
═══════════════════════════════════════
Phase:      {phase} (iteration {n})
Last:       {from} → {to} at {time}
Spec:       {spec_ref or "none"}
Plan:       {plan_ref or "none"}
Gates:      pre-merge: {status} | issue: {status} | re-entry: {status}
Decisions:  {count} logged
Handoffs:   {count} saved
Started:    {started_at or "not started"}
Updated:    {updated_at or "never"}
═══════════════════════════════════════
```

Where gate status is one of: `pending`, `approved`, `rejected`, or `—` (not reached).

### Phase Visualization

Show the loop with the current phase highlighted:

```
SPEC → PLAN → IMPLEMENT → TEST → AUDIT → [GATE] → SHIP → OBSERVE → [GATE] → TRIAGE → [GATE] → ...
                                ▲
                           (you are here)
```

Use `▲` marker under the current phase. If idle, show "Loop is idle — run /sl-spec or /sl-loop to start."
