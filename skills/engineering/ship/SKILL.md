---
name: ship
description: "Run a planned feature's tickets unattended, one after another, in the feature's own worktree."
disable-model-invocation: true
model: sonnet
argument-hint: "<feature-slug>"
---

Run the tickets of the feature `$ARGUMENTS` from start to finish without the user. They planned the tickets and approved them; from here on nobody answers questions. Never ask the user anything: when something blocks you, record it where the steps below say and go on, or stop.

The issue tracker and triage labels should have been provided to you. If not, tell the user to run `/setup-matt-pocock-skills` and stop.

## What the project provides

- `docs/agents/dod.json`, the project's **definition of done**: `environmentChecks` (each with an optional `start` command), `commands`, `implementerAgent` (the subagent that builds a ticket) and `protectedPaths`.
- `.claude/hooks/run-dod.ps1`, which runs part of it and prints `{"ok", "failed"}`. Run it exactly as `pwsh -NoProfile -NonInteractive -File .claude/hooks/run-dod.ps1 -Check environment` (starts what can be started, then reports what still fails) or `... -Check commands` (the DoD commands in order). Exit 0 is green, 1 is red, 2 means the DoD file is missing or unreadable.
- `docs/agents/implementer-contract.md`: what the implementer gets and what it returns.
- Hooks that hold the implementer to that contract: it cannot report success while the DoD is red, and it cannot write a protected path.

If `dod.json` or `run-dod.ps1` is missing, the project is not set up for unattended work: say so and stop.

## Start or resume

Starting and resuming are the same steps. The tracker is the only state: keep no progress anywhere else.

1. **Stay awake.** If the app offers a keep-awake tool, hold the computer awake until the session goes idle.
2. **Enter the feature's worktree.** The feature is built on branch `feat/<feature>` in the worktree `.claude/worktrees/<feature>`, never in the user's checkout.
   - Already in that worktree: go on.
   - `git worktree list` shows it: switch the session into it with the EnterWorktree tool, passing `path`.
   - Otherwise the feature's tickets must be committed, because a new worktree only has what HEAD has: if `git status --porcelain -- .scratch/<feature>` prints anything, stop and say so. Create the worktree from the current HEAD with `git worktree add .claude/worktrees/<feature> -b feat/<feature>` (drop `-b` when the branch already exists), then switch into it with EnterWorktree, passing `path`.
3. **Check the environment** with `run-dod.ps1 -Check environment`.
4. **Check the baseline** with `run-dod.ps1 -Check commands`. The tickets were planned on a green build.

A failure in step 3 or 4, or exit 2 from either, is an **environment failure**: see below.

## Work the tickets

Read every ticket in `.scratch/<feature>/issues/` and pick the next one:

- a ticket whose status is `in-progress` (a run stopped in the middle of it) comes first;
- otherwise the lowest-numbered `ready-for-agent` ticket whose `Blocked by` tickets are all `done`;
- skip every other status. A ticket blocked by one that is not `done` waits.

For the picked ticket:

1. Set `**Status:** in-progress` and commit only that change (`ship: start <NN>`). That commit is the ticket's **base**.
2. Dispatch the implementer subagent named by `implementerAgent`, in the foreground. Pass pointers, never copies: the ticket path, the worktree path, the branch, `round 1`, and the contract `docs/agents/implementer-contract.md`.
3. When it returns, read the first line of its final message and look at the worktree.
   - `STATUS: done`, commits after the base, and `run-dod.ps1 -Check commands` green: the ticket is **done**. If the worktree is not clean, stash the leftovers first (`git stash push --include-untracked -m "ship: <NN> leftovers"`) and say so in the ticket. Set `**Status:** done`, tick the criteria that its `## Comments` entry proves, and commit (`ship: <NN> done`).
   - Anything else is **stuck**: `STATUS: stuck`, a missing status line, no commits after the base, or a red DoD. Put the branch back to green without losing the work: stash any uncommitted changes (`git stash push --include-untracked -m "ship: <NN> stuck"`), then undo the ticket's commits after the base with `git revert --no-edit <base>..HEAD`. Set `**Status:** needs-info`, add the implementer's note under `## Comments` (what it tried, where it stands, what it needs) with the stash name and the reverted commits, and commit (`ship: <NN> needs-info`). Tickets that depend on it now wait; the others go on.

Repeat until no ticket can be picked.

## Environment failure

A check that won't come up (Docker, say), an unreadable DoD file, or a red baseline stops the whole feature: it is not the implementer's to fix. Send a push notification with one line naming the failure (`ship <feature>: environment failure, docker`), then end with the `run-dod.ps1` output.

## End of the run

Report in a few lines: the tickets done, the tickets set to `needs-info` and why, the tickets still waiting, and the branch. Leave the worktree and the branch in place: merging is the user's call.

## Never

- Ask the user anything, or wait for an answer.
- Push, merge into another branch, or delete a branch or a worktree.
- Edit a path in `protectedPaths`, the hooks, `dod.json` or the implementer contract.
- Build in the user's checkout instead of the worktree.
