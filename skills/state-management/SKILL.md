# ShipLoop State Management

You are managing the ShipLoop state machine. The filesystem at `.shiploop/` IS the state. Follow these rules exactly.

## Directory Structure

When initializing a project, create this structure:

```
.shiploop/
├── state.json          # Current loop state (atomic writes only)
├── config.yaml         # Project settings (user-editable)
├── handoffs/           # Session handoff documents
├── specs/              # Active specs
│   └── current.md
├── plans/              # Generated plans
│   └── current.md
├── audits/             # Audit results
├── observations/       # Production telemetry snapshots
└── log/                # Append-only governance log
    └── decisions.jsonl
```

## State File (state.json)

The state file tracks the current position in the loop.

### Valid Phases

```
idle → spec → plan → implement → test → audit → gate_pre_merge
     → ship → observe → gate_issue_detected
     → triage → gate_re_entry → spec → ...
```

### Reading State

Always read `.shiploop/state.json` before any phase operation:

```
Read .shiploop/state.json
```

If the file doesn't exist, the project hasn't been initialized. Run initialization first.

### Writing State (Atomic Writes)

NEVER write directly to `state.json`. Always use atomic writes to prevent corruption:

1. Write the new state to `.shiploop/state.json.tmp`
2. Rename `.shiploop/state.json.tmp` to `.shiploop/state.json`

```bash
# Write to temp file, then atomic rename
cat > .shiploop/state.json.tmp << 'STATEEOF'
{content}
STATEEOF
mv .shiploop/state.json.tmp .shiploop/state.json
```

### State Schema

```json
{
  "version": 1,
  "phase": "idle|spec|plan|implement|test|audit|gate_pre_merge|ship|observe|gate_issue_detected|triage|gate_re_entry",
  "iteration": 0,
  "spec_ref": "specs/current.md",
  "plan_ref": "plans/current.md",
  "gates": {
    "pre_merge": { "decision": "approved|rejected", "at": "ISO-8601", "actor": "human" },
    "issue_detected": null,
    "re_entry": null
  },
  "last_transition": { "from": "test", "to": "audit", "at": "ISO-8601" },
  "started_at": "ISO-8601",
  "updated_at": "ISO-8601"
}
```

## Phase Transitions

### Valid Transitions

| From | To | Trigger |
|------|----|---------|
| idle | spec | `/sl-spec` or `/sl-loop` |
| spec | plan | Spec written to `specs/current.md` |
| plan | implement | Plan written to `plans/current.md` |
| implement | test | Implementation complete (git diff shows changes) |
| test | audit | Tests pass |
| audit | gate_pre_merge | Audit complete |
| gate_pre_merge | ship | Human approves gate |
| gate_pre_merge | spec | Human rejects gate (restart) |
| ship | observe | Deploy verified |
| observe | gate_issue_detected | Issue found |
| observe | idle | No issues (loop complete) |
| gate_issue_detected | triage | Human confirms issue |
| gate_issue_detected | observe | Human dismisses as noise |
| triage | gate_re_entry | Fix spec drafted |
| gate_re_entry | spec | Human approves re-entry |
| gate_re_entry | idle | Human cancels |

### Transition Protocol

On every phase transition:

1. Read current state
2. Validate the transition is valid (check table above)
3. Update `phase`, `last_transition`, `updated_at`
4. If entering a gate phase, clear the gate decision (set to `null`)
5. If a gate is approved, increment `iteration` if looping back to `spec`
6. Write state atomically
7. Log the transition to `decisions.jsonl`

## Decision Log (decisions.jsonl)

Append-only. NEVER overwrite or truncate this file.

### Log Format

One JSON object per line:

```jsonl
{"ts":"2026-02-28T10:30:00Z","phase":"gate_pre_merge","event":"transition","from":"audit","to":"gate_pre_merge","actor":"system"}
{"ts":"2026-02-28T10:31:00Z","phase":"gate_pre_merge","event":"gate_decision","decision":"approved","actor":"human","context":{"tests_passed":42,"security_findings":0}}
```

### Writing to the Log

Always append, never overwrite:

```bash
echo '{"ts":"...", ...}' >> .shiploop/log/decisions.jsonl
```

## Initialization

When `/sl-loop` or `/sl-status` is run in a project without `.shiploop/`:

1. Create the directory structure above
2. Copy `templates/state.json` to `.shiploop/state.json`
3. Copy `templates/config.yaml` to `.shiploop/config.yaml`
4. Create empty directories: `handoffs/`, `specs/`, `plans/`, `audits/`, `observations/`, `log/`
5. Create empty `log/decisions.jsonl`
6. Log initialization event to `decisions.jsonl`

## Handoff Generation

Generate a handoff document at `.shiploop/handoffs/YYYY-MM-DD-HHMM-{description}.md` when:

1. A gate decision is made
2. Before context compaction
3. At session end (if mid-loop)
4. When switching between major phases

### Handoff Template

```markdown
# ShipLoop Handoff — {date} {time}

## What We Accomplished
{completed phases with decision rationale}

## Current State
{active phase, partial work, running processes}

## Lessons Learned
{what worked, mistakes, library quirks}

## Next Steps
- [ ] {prioritized action}
- [ ] {prioritized action}

## Key Files Modified
- `path/to/file`: {what changed}

## Blockers / Open Questions
- {unresolved issues}

## Loop Metadata
- Iteration: {n}
- Gate decisions this session: {list}
```

## Config Reading

Read `.shiploop/config.yaml` for project-specific settings. If it doesn't exist, use defaults from `templates/config.yaml`.

Key settings to check:
- `gates.*.auto_approve` — if true, skip human confirmation at that gate
- `routing.overrides.*` — model tier overrides (agents already have defaults in their frontmatter)
- `handoffs.auto_generate` — whether to auto-generate handoff docs
