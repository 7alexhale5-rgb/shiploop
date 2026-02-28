# Changelog

All notable changes to ShipLoop are documented here.

## [0.5.0] — 2026-02-28

### Fixed
- Resolve 10 P2 issues from self-assessment dogfood:
  - Remove unused `TodoWrite` tool from gate-presenter, quality-reviewer, observer agents
  - Add `.shiploop/` to `.gitignore` for self-dogfooding
  - Fix dead "route to builder" text in boilerplate-gen agent
  - Align config routing keys with actual agent names (`security_auditor`, `gate_presenter`), remove dead keys (`lint_check`, `ship_commands`)
  - Add `loop:` section to config template
  - Reference `spec-template.md` in spec-writer and triage-analyst agents
  - Fix double-increment risk: remove iteration bump from `sl-spec` (owned by `sl-triage`)
  - Add null-safe field access rules to state-management skill
  - Align `sl-audit` blocking definition with gate policy

### Added
- Unknown/corrupted state recovery in state-management skill
- `--force` flag for `/sl-loop` to bypass iteration cap
- Explicit parallelization criteria for scaffold steps in `/sl-build`
- `CHANGELOG.md`

### Changed
- Upgrade gate-presenter agent from haiku to sonnet (high-stakes summary warrants stronger model)

## [0.4.1] — 2026-02-28

### Fixed
- Resolve 6 P0 issues from self-assessment dogfood:
  - Gate decision now persisted to `state.json` so `sl-ship` doesn't block
  - `git diff` no longer misses uncommitted changes in `sl-test` and `sl-audit`
  - Build no longer advances to test on 100% test failures (3-tier check)
  - `--from-observation` reads correct file (`latest-issue.json`)
  - Remove nonexistent `/sl-handoff` from README
  - Add write-scope guardrails to builder and boilerplate-gen agents

- Resolve 13 P1 issues from self-assessment dogfood:
  - Consistent init messaging across all 9 commands
  - Remove unjustified `WebSearch` tool from spec-writer
  - `--force-reentry` now verifies fix spec exists
  - Hard iteration cap in both `sl-triage` and `sl-loop`
  - Standardize `gates.` config namespace
  - `--dry-run` reports no artifacts written
  - Formal 5-step transition block in `sl-observe`
  - Fix `BUILD` → `IMPLEMENT` in phase visualization
  - Configurable merge strategy via `ship.merge_strategy`
  - Agents use provided file list instead of re-running `git diff`
  - Guard `fix-draft.md` copy with existence check
  - Add MCP detection note to observer agent

## [0.4.0] — 2026-02-28

### Added
- Phase 5: Triage + Loop (`/sl-triage`, `/sl-loop`, triage-analyst agent)
- Self-assessment dogfood workflow

## [0.3.0] — 2026-02-28

### Added
- Phase 4: Ship + Observe (`/sl-ship`, `/sl-observe`, ship-ops and observer agents)

## [0.2.0] — 2026-02-28

### Added
- Phase 3: Test/Audit/Gate pipeline (`/sl-test`, `/sl-audit`, `/sl-gate`, 4 agents)
- Phase 2: Inner loop (`/sl-spec`, `/sl-plan`, `/sl-build`, 4 agents)

## [0.1.0] — 2026-02-28

### Added
- Phase 1: Foundation — plugin manifest, state-management skill, `/sl-status`
