---
description: Pre-merge gate — present test/audit summary, await human approval or rejection
argument-hint: Optional --no-pr to skip PR draft creation
---

# /sl-gate

Present a comprehensive gate summary from all test and audit results. The human decides: approve (proceed to ship) or reject (loop back to spec).

## Prerequisites

Follow the **state-management** skill for all state operations in this command.

## Step 1: Validate State

Read `.shiploop/state.json`.

- If `.shiploop/` doesn't exist: tell the user to run `/sl-status --init` or `/sl-loop` to initialize first. Stop.
- Valid transition: `audit → gate_pre_merge`
- If current phase is not `audit`: tell the user the current phase and suggest the appropriate next command
- If current phase is already `gate_pre_merge`: this is a re-review — proceed

## Step 2: Read All Artifacts

Read these files if they exist:
- `.shiploop/audits/latest-tests.json`
- `.shiploop/audits/latest-security.json`
- `.shiploop/audits/latest-quality.json`

Missing files are noted as "not available" — not failures.

## Step 3: Check Gate Policy

Read `.shiploop/config.yaml` `gates.pre_merge` section:
- If `require_tests: true` AND tests failed → mark as **BLOCKER**
- If `require_audit: true` AND high-severity security findings exist → mark as **BLOCKER**
- If `require_clean_audit: true` (from `gates` section) AND any medium+ findings exist → mark as **BLOCKER**

## Step 4: Launch Gate Presenter

Launch the **gate-presenter** agent:
```
Format a pre-merge gate summary from these artifacts:

Test results: .shiploop/audits/latest-tests.json
Security audit: .shiploop/audits/latest-security.json
Quality review: .shiploop/audits/latest-quality.json

Gate policy:
  require_tests: {value}
  require_audit: {value}
  require_clean_audit: {value}

Blockers identified: {list or "none"}
```

## Step 5: Transition State

Transition to `gate_pre_merge`:

1. Update `phase` to `gate_pre_merge`
2. Update `last_transition`
3. Update `updated_at`
4. Write state atomically

Log the transition to `decisions.jsonl`.

**Note:** The gate decision is written to `state.json` in Step 7 after the human decides — not here.

## Step 6: Present Gate + Detect GitHub

Display the formatted gate summary from the gate-presenter agent.

**GitHub MCP detection:**
1. Check if `gh` CLI is available (run `which gh`)
2. Check if in a git repo with a remote (`git remote -v`)
3. If both available AND `config.gates.create_pr_draft: true` AND `--no-pr` was NOT passed:
   - Create a PR draft using `gh pr create --draft --title "{spec title}" --body "{gate summary as markdown}"`
   - Report: "PR draft created: {URL}"
4. If `gh` not available: "Tip: Install GitHub CLI for automatic PR draft creation."
5. If `--no-pr` was passed: skip PR creation silently

## Step 7: Await Decision

If blockers exist, recommend rejection:
"⛔ Gate has {N} blockers. Recommend: reject and address findings."

If no blockers, recommend approval:
"✅ Gate is clean. Recommend: approve to proceed to ship."

Ask the user to choose:
- **Approve** → transition to `ship` phase. Log: `{decision: "approve", actor: "human", blockers: N, advisory: M}`
- **Reject** → transition to `spec` phase (loop back). Log: `{decision: "reject", actor: "human", reason: "{user's reason}"}`. Increment iteration counter in state.

**Write gate decision to state.json:**
After the human decides, update `state.json`:
- Set `gates.pre_merge.decision` to `"approved"` or `"rejected"`
- Set `gates.pre_merge.actor` to `"human"`
- Set `gates.pre_merge.decided_at` to current ISO-8601 timestamp
- Write state atomically (this is what `/sl-ship` reads to confirm approval)

After decision:
```
{If approved:}
Gate approved → proceeding to ship
Phase: gate_pre_merge → ship

{If rejected:}
Gate rejected → looping back to spec
Phase: gate_pre_merge → spec (iteration {N+1})
Reason: {user's reason}

Next: Update the spec with /sl-spec to address findings, then re-plan with /sl-plan.
```