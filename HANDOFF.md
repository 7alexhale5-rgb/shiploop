# ShipLoop — Self-Assessment Handoff

**Date**: 2026-02-27 ~9:00 PM
**Status**: Phase 5 complete — plugin is feature-complete (v0.4.0)
**Repo**: `~/Projects/shiploop/` (5 commits + Phase 5 uncommitted, 33 files, ~2,570 LOC)

---

## Mission: Dogfood ShipLoop Against Itself

Use ShipLoop's own commands to assess and evaluate ShipLoop. The plugin is a collection of markdown files (commands, agents, skills, templates) — treat it as a codebase and run the lifecycle against it.

### What to do

1. **`/sl-loop`** — Initialize `.shiploop/` in the shiploop repo and start the lifecycle
2. **`/sl-spec`** — Write a spec for "ShipLoop v0.4.0 self-assessment": evaluate command quality, agent design, state machine correctness, config completeness, and documentation coverage
3. **`/sl-plan`** — Generate an assessment plan from the spec
4. **`/sl-build`** — Execute the plan (in this case: produce an assessment report as a markdown artifact)
5. **`/sl-test`** — Validate the assessment by cross-checking claims against actual files
6. **`/sl-audit`** — Security + quality review of all ShipLoop files
7. **`/sl-gate`** — Present findings at the gate for human review

The goal is NOT to ship a release — it's to stress-test the plugin's own workflow on a real (meta) task and identify gaps, friction, or broken flows.

### What to evaluate

| Area | Questions |
|------|-----------|
| **Commands** | Do all 11 commands follow the 7-step pattern consistently? Are prerequisites, state validation, and transitions correct? |
| **Agents** | Are tool permissions appropriate? Are model tiers justified? Do agent outputs match what commands expect? |
| **State machine** | Are all transitions in SKILL.md reachable? Are there dead states or impossible transitions? |
| **Config** | Does config.yaml cover all configurable behaviors mentioned in commands? Any missing knobs? |
| **Graceful degradation** | Do commands handle missing MCPs, missing artifacts, wrong-state errors helpfully? |
| **Documentation** | Is README.md accurate? Do commands have clear argument-hints? Is the handoff template complete? |
| **Edge cases** | What happens at iteration cap? What if specs/current.md is missing? What if state.json is corrupted? |

### Expected output

A structured assessment report with:
- Per-area findings (what works, what's broken, what's missing)
- Severity ratings (P0 = broken flow, P1 = poor UX, P2 = improvement, P3 = nice-to-have)
- Specific file:line references for each finding
- Recommended fixes (as potential specs for future iterations)

---

## Current File Map (33 files)

```
shiploop/
├── .claude-plugin/
│   └── plugin.json                    # v0.4.0 — 11 commands, 11 agents, 1 skill
├── agents/
│   ├── spec-writer.md                 # sonnet — structures feature specs
│   ├── planner.md                     # opus — generates implementation plans
│   ├── builder.md                     # sonnet — executes plan steps
│   ├── boilerplate-gen.md             # haiku — config scaffolding
│   ├── test-generator.md              # sonnet — JiTTest generation
│   ├── security-auditor.md            # opus — OWASP/SAST (read-only)
│   ├── quality-reviewer.md            # sonnet — code quality (read-only)
│   ├── gate-presenter.md              # haiku — formats gate summary
│   ├── ship-ops.md                    # haiku — git tag/push/merge (no Write/Edit)
│   ├── observer.md                    # sonnet — Sentry queries + deploy health
│   └── triage-analyst.md              # sonnet — root cause analysis (read-only + .shiploop/ writes)
├── commands/
│   ├── sl-status.md                   # Show state, phase, transitions
│   ├── sl-spec.md                     # Capture/refine feature spec
│   ├── sl-plan.md                     # Generate plan from spec
│   ├── sl-build.md                    # Execute plan (builder + boilerplate-gen)
│   ├── sl-test.md                     # JiTTest — generate + run tests
│   ├── sl-audit.md                    # Security + quality review
│   ├── sl-gate.md                     # Pre-merge gate (human approve/reject)
│   ├── sl-ship.md                     # Tag, merge PR, push (deployment-agnostic)
│   ├── sl-observe.md                  # Post-deploy monitoring (Sentry/manual)
│   ├── sl-triage.md                   # Multi-gate: confirm issue + approve re-entry
│   └── sl-loop.md                     # Master orchestrator — state→command routing
├── skills/
│   └── state-management/
│       └── SKILL.md                   # State machine, transitions, decision log
├── templates/
│   ├── config.yaml                    # Default project config (all sections)
│   ├── state.json                     # Initial state template
│   └── spec-template.md              # Spec structure template
├── README.md                          # Plugin overview + installation
└── HANDOFF.md                         # This file
```

## State Machine (Complete)

```
idle → spec → plan → implement → test → audit → gate_pre_merge
  → ship → observe → idle (clean)
  → observe → gate_issue_detected → triage → gate_re_entry → spec (loop back)
  → gate_issue_detected → observe (dismiss)
  → gate_re_entry → idle (cancel)
  → gate_pre_merge → spec (reject)
```

## Agent Model Tiers

| Tier | Agents | Rationale |
|------|--------|-----------|
| **opus** | planner, security-auditor | High-stakes decisions — precision prevents cascading errors |
| **sonnet** | spec-writer, builder, test-generator, quality-reviewer, observer, triage-analyst | Standard implementation + analysis work |
| **haiku** | boilerplate-gen, ship-ops, gate-presenter | Mechanical/formatting tasks — no judgment needed |

## Config Sections (templates/config.yaml)

`routing` · `gates` · `test` · `audit` · `gate` · `ship` · `observe` · `triage` · `mcp` · `handoffs`

## Known Gaps (Pre-assessment)

These are suspected but unverified — the self-assessment should confirm or deny:
1. README.md may be out of date (written during Phase 2, now at Phase 5)
2. No explicit error messages for corrupted state.json
3. `/sl-loop` doesn't handle `--force` to bypass iteration cap
4. No `CHANGELOG.md` or version history
5. Config doesn't have a `loop:` section (orchestrator settings live in `triage:`)

---

## How to Start

```
cd ~/Projects/shiploop
# Run /sl-loop to initialize and begin the self-assessment lifecycle
```

The spec should describe what "assessing ShipLoop" means — the plan will break it into reviewable chunks — and the build will produce the actual assessment report. Let the plugin drive.
