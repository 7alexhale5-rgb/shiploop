# ShipLoop — Post-Assessment Handoff

**Date**: 2026-02-28
**Status**: v0.4.1 — all P0s and P1s resolved, P2s remain
**Repo**: `~/Projects/shiploop/` (8 commits, 33 files, ~2,650 LOC)

---

## What Was Done This Session

### Self-Assessment Dogfood
Ran 3 parallel review agents (commands, agents, config/templates) to produce a structured assessment of the entire plugin. Found **6 P0s, 13 P1s, 10 P2s, and 5 P3s**.

### P0 Fixes (commit `27db1ce`)
| # | Issue | Fix |
|---|-------|-----|
| 1 | Gate decision never persisted to `state.json` — `sl-ship` always blocked | Added `gates.pre_merge.decision/actor/decided_at` write to `sl-gate.md` Step 7 |
| 2 | `git diff HEAD` missed uncommitted changes — tests/audits ran on empty | Removed `HEAD` suffix in `sl-test.md` and `sl-audit.md` |
| 3 | Build advanced to `test` even with 100% failures | Added 3-tier check: all-fail→stop, partial→confirm, all-pass→advance |
| 4 | `--from-observation` read wrong file (`latest.json` vs `latest-issue.json`) | Fixed path in `sl-spec.md` |
| 5 | `/sl-handoff` in README but no command file exists | Removed from README |
| 6 | Builder + boilerplate-gen had no write-scope guardrail | Added write-scope rules to both agents |

### P1 Fixes (commit `ed8a5b6`)
| # | Issue | Fix |
|---|-------|-----|
| 1 | Inconsistent init messaging (only `/sl-status --init`) | All 9 commands now mention both `--init` and `/sl-loop` |
| 2 | `spec-writer` had unjustified `WebSearch` tool | Removed from tools line |
| 3 | `--force-reentry` didn't verify fix spec exists | Added artifact existence check |
| 4 | Iteration cap: soft prompt in `sl-triage` vs hard stop in `sl-loop` | Changed to hard stop (consistent) |
| 5 | `gate.` vs `gates.` config namespace | Standardized to `gates.` |
| 6 | `--dry-run` report referenced nonexistent artifacts | Added "no artifacts written" note |
| 7 | `sl-observe` transition was informal one-liner | Replaced with formal 5-step transition block |
| 8 | Phase visualization showed `BUILD` instead of `IMPLEMENT` | Fixed in `sl-status.md` |
| 9 | Ship-ops hardcoded `--squash` merge strategy | Made configurable via `ship.merge_strategy` |
| 10 | Agents redundantly re-ran `git diff` | Changed to "use provided file list" |
| 11 | `fix-draft.md` copy with no existence guard | Added guard in `sl-triage.md` Step 5 |
| 12 | Sentry MCP Path A was a dead branch | Added MCP detection note to observer agent |

---

## What's Next — Remaining P2s

| # | Issue | File(s) |
|---|-------|---------|
| 1 | Config routing key names don't match agent names | `templates/config.yaml` |
| 2 | Double-increment risk: both `sl-triage` and `sl-spec` increment iteration | `commands/sl-triage.md`, `commands/sl-spec.md` |
| 3 | `state.json` null refs not guarded in downstream commands | Multiple commands |
| 4 | No `loop:` section in config | `templates/config.yaml` |
| 5 | `TodoWrite` granted to agents that never use it | `agents/gate-presenter.md`, `agents/quality-reviewer.md`, `agents/observer.md` |
| 6 | `spec-template.md` not referenced by spec-producing agents | `agents/spec-writer.md`, `agents/triage-analyst.md` |
| 7 | Triage fix spec uses different schema than spec-writer | `agents/triage-analyst.md` |
| 8 | `.gitignore` missing `.shiploop/` for self-dogfooding | `.gitignore` |
| 9 | Audit "blocking" definition diverges from gate policy | `commands/sl-audit.md` |
| 10 | `boilerplate-gen` "route to builder" instruction is dead text | `agents/boilerplate-gen.md` |

### P3s (nice-to-have)
- No `CHANGELOG.md`
- No corrupted/unknown state handling
- No `--force` flag for `sl-loop` iteration cap
- Build parallelization exception underspecified
- Gate presenter is haiku for high-stakes summary

---

## Beyond Bug Fixes

Once P2s are resolved, the plugin is ready for:
1. **Marketplace publish** — needs `CHANGELOG.md`, final README review
2. **Real-world dogfood** — use ShipLoop on an actual project (consult-ops, contract-extract, etc.)
3. **GitHub integration** — push to a public/private repo for distribution

---

## Git Log

```
ed8a5b6 fix: resolve 13 P1 issues from self-assessment dogfood
27db1ce fix: resolve 6 P0 issues from self-assessment dogfood
e6c72e0 feat: add Phase 5 triage + loop — 2 commands + 1 agent, self-assessment handoff
91a2bb0 feat: add Phase 4 ship + observe — 2 commands + 2 agents
7fb43f4 chore: upgrade planner agent to opus for precision
96e8524 feat: add Phase 3 test/audit/gate pipeline — 3 commands + 4 agents
4f175db feat: add Phase 2 inner loop — /sl-spec, /sl-plan, /sl-build + 4 agents
2d251ff feat: scaffold Phase 1 foundation — plugin manifest, state management, /sl-status
```
