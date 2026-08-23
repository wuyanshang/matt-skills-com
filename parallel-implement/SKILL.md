---
name: parallel-implement
description: Implement an approved set of Markdown tickets in parallel with OpenCode subagents, respecting Blocked by dependencies, local branch isolation, test gates, merge checkpoints, and resumable run state. Use when /to-tickets has produced approved tickets and the user wants the implementation executed automatically.
disable-model-invocation: true
argument-hint: "Path to the approved ticket directory"
---

# Parallel Implement

Execute an already-approved ticket graph. This skill does not invent, split, or rewrite requirements. Planning belongs to `/to-tickets`; this skill consumes its output.

## Inputs

Accept a ticket directory as the argument. Each ticket is a Markdown file containing a title, `What to build`, `Blocked by`, `Status`, and acceptance criteria. The current repository is the integration worktree.

If no directory is supplied:

1. Look for `.scratch/*/issues/` directories.
2. Continue only when exactly one plausible ticket directory exists.
3. Otherwise ask the user for the ticket directory. Do not regenerate tickets.

Read the related spec and repository instructions before dispatching work. Pass paths to subagents instead of copying large documents into their prompts.

## Operating rules

- The dependency graph is authoritative. Never start a ticket while any `Blocked by` ticket is incomplete.
- The frontier is the set of tickets whose blockers are complete. Dispatch only frontier tickets.
- Run at most three implementers concurrently.
- Independent tickets may continue when another ticket fails. Descendants of a failed ticket remain waiting.
- The integration branch is local. Never push, create a remote PR, or alter a parent issue.
- Each implementer owns exactly one ticket and its own local branch/worktree.
- Subagents must not merge, edit another ticket, or delete worktrees.
- The coordinator is the only process that changes run state, ticket status, or the integration branch.

## Run state

Create a state directory at the repository root:

```text
.parallel-implement/<feature-slug>/run-state.json
```

Record, for every ticket:

```json
{
  "id": "T-001",
  "title": "...",
  "status": "pending | running | completed | failed | blocked | interrupted",
  "blocked_by": [],
  "branch": "parallel/<feature>/<ticket-id>",
  "worktree": "...",
  "subagent_session": null,
  "commits": [],
  "tests": [],
  "last_error": null,
  "updated_at": "..."
}
```

Write the state after every dispatch, completion, failure, merge, and cleanup. Treat Git and this file as the source of truth, not a lost subagent transcript.

## Preflight

1. Resolve the ticket directory and read every ticket completely.
2. Parse IDs, titles, `Blocked by`, acceptance criteria, and statuses.
3. Validate that every blocker exists and that the graph has no cycle. Stop on malformed or ambiguous dependencies.
4. Identify the current branch as the integration branch.
5. Require a clean integration worktree. Do not stash, reset, or discard user changes automatically.
6. Create the state file and a sibling worktree root, for example:

```text
<repository>-parallel-worktrees/<feature>/<ticket-id>
```

Use local branches such as `parallel/<feature>/<ticket-id>`. Branches and worktrees are temporary implementation isolation, not deliverables.

## Dispatch

For each frontier ticket, create its branch/worktree from the current integration branch and start one OpenCode implementer subagent. Give it:

- the repository instructions;
- the spec path;
- the ticket path;
- its worktree and branch;
- the acceptance criteria;
- the required test commands discovered from project configuration;
- the completion report format below.

The implementer should explore the relevant code, use `/tdd` at the agreed seam where practical, run focused checks regularly, and commit its work locally. It may create checkpoint commits so interrupted work can resume.

Required completion report:

```text
ticket: <id>
status: completed | failed | blocked
branch: <local branch>
worktree: <path>
commits: <hashes>
changed_scope: <modules or files>
tests: <commands and results>
blockers: <remaining issues, or none>
```

A natural-language claim of completion is not enough. Verify the branch, diff, commit, and test output from the coordinator.

## Verification and merge

When an implementer reports completion:

1. Confirm the worktree has no unexpected uncommitted changes.
2. Check that the diff stays within the ticket's scope.
3. Run the ticket's focused tests and typecheck/build checks.
4. If all gates pass, integrate the ticket into the integration branch as one ticket-level commit, preserving the ticket ID in the commit message. Squash checkpoint commits when practical.
5. Update the state and ticket status only from the coordinator.
6. Remove the temporary worktree after a successful merge. Keep the local branch until the run is verified, then clean it up.
7. Recompute the frontier and dispatch newly unblocked tickets.

Clean, mechanical merges may be performed by the coordinator. Do not guess through a semantic conflict. Pause and use `/resolving-merge-conflicts` or a dedicated resolver subagent with the spec and both tickets as primary sources.

## Failures and recovery

- Retry an environment or command failure once.
- For a clear test failure, return the output to the original implementer for one repair attempt.
- After a second failure, mark the ticket failed and pause descendants. Continue independent frontier tickets.
- Stop for contradictory requirements, missing acceptance criteria, scope expansion, or unresolved semantic conflicts.
- Never silently change acceptance criteria or dependency edges.

When resuming after the main session or a subagent was interrupted:

1. Load `run-state.json`.
2. Treat stale `running` entries as `interrupted`.
3. Inspect each recorded branch and worktree with Git.
4. If a completed commit and checks exist, verify and merge it.
5. If uncommitted or checkpoint work exists, start a recovery subagent in the same worktree with the ticket and state paths.
6. If no work exists, start a fresh implementer.
7. Do not launch descendants until their blockers are actually merged into the integration branch.

Never delete an interrupted worktree before inspecting it.

## Finalization

When every ticket is completed and merged:

1. Run the full test suite, typecheck, and build commands.
2. Invoke `/code-review` on the complete integration diff.
3. Fix review findings in a focused repair subagent, then rerun the affected checks.
4. Report the integration branch, ticket commits, tests, review result, and any remaining local branches/worktrees.

Do not push or claim completion while required checks or review findings remain unresolved.

## Invocation example

```text
/parallel-implement .scratch/order-flow/issues
```

The user invokes `/to-tickets` separately, approves its breakdown, then starts this skill in a fresh session. The ticket files are the handoff boundary between planning and execution.
