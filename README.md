# ShipLoop

Complete dev lifecycle orchestration for Claude Code. SPEC → PLAN → BUILD → TEST → AUDIT → SHIP → OBSERVE → TRIAGE — with human gates at critical checkpoints.

## What It Does

ShipLoop is a Claude Code plugin that runs the full development lifecycle as a continuous loop. Every phase is automated; you only intervene at 3 gates:

1. **Pre-merge** — review test results, audit findings, and diff before merging
2. **Issue detected** — confirm a production observation warrants action
3. **Re-entry** — approve a fix spec before the loop restarts

Between gates, ShipLoop handles spec writing, planning, implementation, testing, security auditing, shipping, and production observation automatically.

## Install

```bash
# From Claude Code marketplace (when published)
/plugin marketplace add shiploop

# Local development
/plugin add ~/Projects/shiploop
```

## Commands

| Command | Description |
|---------|-------------|
| `/sl-status` | Show current loop state and phase |
| `/sl-spec` | Generate or refine a spec |
| `/sl-plan` | Break spec into implementation plan |
| `/sl-build` | Execute plan with subagent-driven development |
| `/sl-test` | JiTTest — generate ephemeral per-change tests |
| `/sl-audit` | Security scan + quality review |
| `/sl-gate` | Present gate summary, await approval |
| `/sl-ship` | Merge, deploy, verify health |
| `/sl-observe` | Surface production issues |
| `/sl-triage` | Auto-generate fix spec from observation |
| `/sl-loop` | Run the full loop automatically |

## How It Works

The filesystem is the state machine. ShipLoop creates a `.shiploop/` directory in your project with:

- `state.json` — current phase and loop metadata
- `config.yaml` — project-level settings
- `specs/` / `plans/` — active spec and plan
- `audits/` — security and quality results
- `log/decisions.jsonl` — append-only governance trail
- `handoffs/` — session handoff documents

## Model Routing

Agents use the right model for the job:

- **Opus** — security audits (high-stakes, deep reasoning)
- **Sonnet** — spec writing, planning, building, testing, triage
- **Haiku** — boilerplate generation, gate formatting, ship commands

## MCP Integration

ShipLoop orchestrates existing MCPs. If an MCP isn't installed, the relevant command gracefully degrades — showing manual alternatives instead of failing.

| MCP | Used By | Without It |
|-----|---------|------------|
| GitHub | `/sl-ship`, `/sl-gate` | Manual git commands shown |
| Playwright | `/sl-test` | Unit tests only, skip E2E |
| Context7 | `/sl-build` | Falls back to built-in knowledge |
| Sentry | `/sl-observe` | Manual observation input |
| Linear | `/sl-observe` | Manual observation input |

## License

MIT