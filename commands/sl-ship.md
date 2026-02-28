---
description: Tag release, merge PR, push to remote — deployment-agnostic shipping
argument-hint: Optional --dry-run to preview operations without executing
---

# /sl-ship

Ship the approved code by creating a release tag, merging the PR (if exists), and pushing to remote. ShipLoop is deployment-agnostic — actual deployment is handled by your CI/CD pipeline.

## Prerequisites

Follow the **state-management** skill for all state operations in this command.

## Step 1: Validate State

Read `.shiploop/state.json`.

- If `.shiploop/` doesn't exist: tell the user to run `/sl-status --init` first. Stop.
- Valid transition: `gate_pre_merge → ship`
- The gate must show `decision: "approved"` — if gate was rejected, the state should be `spec`, not here.
- If current phase is not `gate_pre_merge`: tell the user the current phase and suggest the appropriate next command.

## Step 2: Parse Input

Check `$ARGUMENTS` for flags:
- `--dry-run`: Show what operations would be performed without executing them.

## Step 3: Read Gate Result

Read `.shiploop/state.json` to confirm gate approval:
- `gates.pre_merge.decision` must be `"approved"`
- `gates.pre_merge.actor` must be `"human"`

If gate is not approved, stop: "Gate was not approved — cannot ship."

Read config from `.shiploop/config.yaml`:
- `ship.tag_format` (default: `v{iteration}.{date}`)
- `ship.merge_pr` (default: `true`)
- `ship.push_remote` (default: `origin`)
- `ship.push_branch` (default: `main`)

Compute the tag name by replacing `{iteration}` with the state's iteration number and `{date}` with today's date (YYYY-MM-DD).

## Step 4: Launch Ship Ops

If `--dry-run`:
```
Dry run — would execute:
  1. git tag -a "{tag}" -m "ShipLoop release {tag}"
  2. gh pr merge --squash (if PR exists and merge_pr: true)
  3. git push {remote} {branch} --tags
```
Skip to Step 7.

Otherwise, launch the **ship-ops** agent:
```
Execute shipping operations:

Tag: {computed tag name}
Remote: {push_remote}
Branch: {push_branch}
Merge PR: {merge_pr}
Iteration: {iteration}
Date: {date}
```

## Step 5: Verify Deploy Signal

After ship-ops completes successfully:

1. Check if `gh` is available: `which gh`
2. If available, check CI status: `gh run list --limit 3`
   - If a workflow was triggered by the push, note its status
   - If CI is failing, warn the user but continue to observe phase
3. If `gh` not available, tell the user: "Verify your CI/CD pipeline picked up the push, then continue with `/sl-observe`."

## Step 6: Write Ship Artifact

Write `.shiploop/audits/latest-ship.json`:
```json
{
  "generated_at": "ISO-8601 timestamp",
  "tag": "{computed tag}",
  "commit": "{current HEAD short SHA}",
  "pr_url": "{PR URL if merged, null if none}",
  "pr_merged": true|false,
  "remote": "{push_remote}",
  "branch": "{push_branch}"
}
```

## Step 7: Transition + Report

1. Transition state: `gate_pre_merge → ship`
2. Log to `decisions.jsonl`: ship operation details, tag, commit
3. Report to user:

```
Shipped → {tag}

Tag: {tag} (pushed to {remote}/{branch})
Commit: {short SHA}
PR: merged (#42) | no open PR | gh not available
CI: triggered (workflow #{id}) | not detected

Next: Run /sl-observe to monitor for post-deploy issues.
```

If `--dry-run`: "Dry run complete. Run `/sl-ship` without `--dry-run` to execute."
