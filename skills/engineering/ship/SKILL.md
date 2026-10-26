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
- `.claude/hooks/check-diff.ps1`, which the verifier runs over a ticket's commits: protected paths changed, analyzer suppressions added, projects dropped from the solution, existing tests changed.
- `docs/agents/implementer-contract.md`: what the implementer gets and what it returns.
- Hooks that hold the implementer to that contract: it cannot report success while the DoD is red, and it cannot write a protected path.

If `dod.json`, `run-dod.ps1` or `check-diff.ps1` is missing, the project is not set up for unattended work: say so and stop.

Three subagents do the work, each in its own role: the implementer named by `implementerAgent` builds, `mattpocock-skills:verifier` (Haiku) checks the definition of done and the evidence, `mattpocock-skills:spec-reviewer` (Sonnet) checks the change against the ticket. Pass each of them pointers, never copies, and wait for each report, doing nothing else meanwhile (where the harness runs subagents in the background, that means waiting for its completion notice).

## Start or resume

Starting and resuming are the same steps. The tracker is the only state: keep no progress anywhere else. Run the session in auto mode, never with permission checks bypassed: the hooks and the allowlist are the guard rails, not your judgement.

1. **Stay awake.** If the app offers a keep-awake tool, hold the computer awake until the session goes idle.
2. **Enter the feature's worktree.** The feature is built on branch `feat/<feature>` in the worktree `.claude/worktrees/<feature>`, never in the user's checkout.
   - Already in that worktree: go on.
   - `git worktree list` shows it: switch the session into it with the EnterWorktree tool, passing `path`.
   - Otherwise the feature's tickets must be committed, because a new worktree only has what HEAD has: if `git status --porcelain -- .scratch/<feature>` prints anything, stop and say so. Create the worktree from the current HEAD with `git worktree add .claude/worktrees/<feature> -b feat/<feature>` (drop `-b` when the branch already exists), then switch into it with EnterWorktree, passing `path`.
   - In the worktree, uncommitted changes with no ticket `in-progress` are leftovers of a stopped run: stash them (`git stash push --include-untracked -m "ship: leftovers"`) and name the stash in your final report. With a ticket `in-progress` they are its work: leave them for the implementer.
3. **Check the environment** with `run-dod.ps1 -Check environment`.
4. **Check the baseline** with `run-dod.ps1 -Check commands`. The tickets were planned on a green build.

A failure in step 3 or 4, or exit 2 from either, is an **environment failure**: see below.

## Work the tickets

Read every ticket in `.scratch/<feature>/issues/` and pick the next one:

- a ticket whose status is `in-progress` (a run stopped in the middle of it) comes first;
- otherwise the lowest-numbered `ready-for-agent` ticket whose `Blocked by` tickets are all `done`;
- skip every other status. A ticket blocked by one that is not `done` waits.

For the picked ticket, first fix its **base**: set `**Status:** in-progress` and commit only that change (`ship: start <NN>`). A ticket that is already `in-progress` keeps its base: the latest `ship: start <NN>` commit on the branch (`git log -1 --grep "ship: start <NN>"`), or the current HEAD when there is none; commit nothing for it.

### Rounds

A ticket gets at most **three rounds**. Its findings files, `.scratch/<feature>/findings/<NN>-round-<r>.md`, count the rounds already spent, so a resumed ticket goes on from the next one; one that already has three is **stuck**. Each round:

1. **Build.** Note HEAD, then dispatch the implementer with the ticket path, the worktree path, the branch, the round number, the contract `docs/agents/implementer-contract.md`, and from round 2 the previous round's findings file. Round 3 runs the implementer on Opus (the Agent tool's `model: opus`): a ticket the Sonnet rounds couldn't land gets the stronger model before the loop gives up on it.
2. **Read its report**, once the implementer has really stopped: if it starts working again after reporting, the DoD gate has sent it back, so wait for its next report.
   - `STATUS: stuck` in round 1 or 2: go straight to round 3 on Opus, passing the implementer's note as the findings. If the note says the ticket needs a decision from the user or a change to a protected path, no model can help: the ticket is **stuck**. `STATUS: stuck` in round 3, or a missing status line: the ticket is **stuck**.
   - Round 1 with no commits after the base: the ticket is **stuck**.
   - Otherwise, stash anything uncommitted (`git stash push --include-untracked -m "ship: <NN> round <r> leftovers"`), so the verifier's DoD run and its diff check see the same committed state.
3. **Check the environment** with `run-dod.ps1 -Check environment`. A failure here is an environment failure, not the ticket's fault.
4. **Verify.** Dispatch the verifier with the worktree path, the ticket path and the base. It answers `VERDICT: pass` (with the criteria it confirmed), `VERDICT: fail` (with findings) or `VERDICT: error` (a script could not run: treat it as an environment failure).
5. **Review.** When the verifier passes, dispatch the spec reviewer with the ticket path, the base, the ticket's earlier findings files and the out-of-scope tickets already filed from it, so it doesn't report the same thing twice.
6. **Decide.**
   - No in-scope findings from either: the ticket is **done**.
   - In-scope findings: write them, numbered and with their source, to the round's findings file and commit it (`ship: <NN> findings round <r>`). After round 1 or 2, start the next round. After round 3 the ticket is **stuck**.
   - Out-of-scope findings never fail a round: each becomes a new ticket (below).

A verifier finding is never waived: a protected path changed, a suppression added, a weakened test, a project dropped from the solution or a red DoD always counts as in-scope, however good the rest is. The implementer may not touch a protected path, even to undo its own change, so put such a path back yourself before the next round: `git checkout <base> -- <path>`, then commit (`ship: <NN> restore <path>`). Say so in the findings file.

### Done

Set `**Status:** done`, tick the criteria the verifier confirmed, name any leftovers stash in the ticket, and commit (`ship: <NN> done`).

### Stuck

Put the branch back to green without losing the work. Stash anything uncommitted (`git stash push --include-untracked -m "ship: <NN> stuck"`). Then undo the implementer's commits after the base, newest first, keeping your own `ship:` commits (findings files, restored paths, new tickets): `git revert --no-edit` on the commits that `git log --format=%h --invert-grep --grep "^ship: " <base>..HEAD` lists, in that order. Set `**Status:** needs-info` and add a **brief** under `## Comments`, short enough to read on a phone:

- **Tried:** the rounds spent and what each implementer did, in a line each.
- **Where it stands:** the last findings that were not resolved, the stash name, the reverted commits.
- **Needs:** the decision or information that would unblock it.

Commit (`ship: <NN> needs-info`) and send a push notification (the harness's push-notification tool, if it has one): `ship <feature>: ticket <NN> needs info, see its Comments`. Tickets that depend on it now wait; the others go on.

### Out-of-scope findings

Each one becomes a ticket in the same `issues/` folder, with the next free number, `**Status:** needs-triage` and `**Blocked by:** None`. Its body is the finding and where it came from (ticket `<NN>`, round `<r>`). Skip a finding that repeats a ticket already filed from the same source ticket. Commit them (`ship: <NN> out-of-scope findings`). `needs-triage` is not `ready-for-agent`, so this run never picks them up: they wait for the user's triage and block nothing.

Repeat until no ticket can be picked.

## Environment failure

A check that won't come up (Docker, say), an unreadable DoD file, or a red baseline stops the whole feature: it is not the implementer's to fix. Send a push notification (the harness's push-notification tool, if it has one) with one line naming the failure (`ship <feature>: environment failure, docker`), then end with the `run-dod.ps1` output.

## End of the run

Report in a few lines: the tickets done, the tickets set to `needs-info` and why, the new `needs-triage` tickets, the tickets still waiting, and the branch. Leave the worktree and the branch in place: merging is the user's call.

## Never

- Ask the user anything, or wait for an answer.
- Waive a finding, or fix a ticket yourself instead of sending it back to the implementer.
- Push, merge into another branch, or delete a branch or a worktree.
- Edit a path in `protectedPaths`, the hooks, `dod.json` or the implementer contract. Putting one back from the base, as the rounds describe, is the only write allowed there.
- Build in the user's checkout instead of the worktree.
