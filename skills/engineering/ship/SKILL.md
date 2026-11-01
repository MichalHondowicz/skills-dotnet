---
name: ship
description: "Run a planned feature's tickets unattended, one after another, in the feature's own worktree, and merge it into main on the user's OK at gate 2."
disable-model-invocation: true
model: sonnet
effort: medium
argument-hint: "<feature-slug>"
---

Run the tickets of the feature `$ARGUMENTS` from start to finish without the user. They planned the tickets and approved them; from here on nobody answers questions until the feature is ready. Never ask the user anything but the one question of *Gate 2*: when something blocks you, record it where the steps below say and go on, or stop.

The issue tracker and triage labels should have been provided to you. If not, tell the user to run `/setup-matt-pocock-skills` and stop.

## What the project provides

- `docs/agents/dod.json`, the project's **definition of done**: `environmentChecks` (each with an optional `start` command), `commands`, `implementerAgent` (the subagent that builds a ticket), `protectedPaths` and `quota` (`weeklyPercent` and `fiveHourPercent`, the plan usage at which no ticket starts). Optionally `standardsSkill`, the project's own review of its coding standards, which the branch review runs, and `screenPaths`, globs of the files that make up screens, which the brief lists.
- `.claude/hooks/run-dod.ps1`, which runs part of it and prints `{"ok", "failed"}`. Run it exactly as `pwsh -NoProfile -NonInteractive -File .claude/hooks/run-dod.ps1 -Check environment` (starts what can be started, then reports what still fails; it also prints `started`, the checks it had to start), `... -Check commands` (the DoD commands in order; it also prints `passed`, the last lines of each command that passed) or `... -Check quota -FiveHourPercent <n> -WeeklyPercent <n>` (the usage you read against `quota`; each window at or above its threshold is `failed`, named `weekly` or `fiveHour`). Exit 0 is green, 1 is red, 2 means the DoD file is missing or unreadable, or a quota check lacks its numbers.
- `.claude/hooks/wait-for-reset.ps1`, run exactly as `pwsh -NoProfile -NonInteractive -File .claude/hooks/wait-for-reset.ps1 -Until <resetsAt>`: it holds the computer awake and returns two minutes after that time. Exit 2 means the time was unreadable or more than six hours ahead.
- `.claude/hooks/check-diff.ps1`, which the verifier runs over a ticket's commits: protected paths changed, analyzer suppressions added, projects dropped from the solution, existing tests changed.
- `docs/agents/implementer-contract.md`: what the implementer gets and what it returns.
- Hooks that hold the implementer to that contract: it cannot report success while the DoD is red, and it cannot write a protected path.

If `dod.json`, `run-dod.ps1`, `wait-for-reset.ps1` or `check-diff.ps1` is missing, the project is not set up for unattended work: say so and stop.

Three subagents do the work, each in its own role: the implementer named by `implementerAgent` builds, `mattpocock-skills:verifier` (Haiku) checks the definition of done and the evidence, `mattpocock-skills:spec-reviewer` (Sonnet) checks the change against the ticket. After the last ticket two reviewers on Opus look at the whole branch once: `mattpocock-skills:standards-reviewer` and the spec reviewer again. Pass each of them pointers, never copies, and wait for each report, doing nothing else meanwhile (where the harness runs subagents in the background, that means waiting for its completion notice).

## Start or resume

Starting and resuming are the same steps. The tracker is the only state: keep no progress anywhere else. Run the session in auto mode, never with permission checks bypassed: the hooks and the allowlist are the guard rails, not your judgement. Run every command this skill names as a plain command of its own, never joined with `;`, `&&` or a pipe or wrapped in `$(...)`: a session isolated in a worktree refuses a compound command that runs `git`, and the tool reports the exit code anyway. Run the project's scripts (`pwsh -NoProfile -NonInteractive -File .claude/hooks/<script>.ps1 ...`) with the PowerShell tool where the session has one: a worktree session's Bash tool refuses any command that runs `pwsh`, even a plain one.

1. **Stay awake.** If the app offers a keep-awake tool, hold the computer awake until the session goes idle.
2. **Note Remote Control.** Read this session's `remoteControlState` with the app's session tool (`get_session` with `"self"`; load it with ToolSearch if it is deferred). A push reaches the user's phone only while it is `on`, so name the state in your final report. Never switch it yourself: the tool is the user's to use.
3. **Enter the feature's worktree.** The feature is built on branch `feat/<feature>` in the worktree `.claude/worktrees/<feature>`, never in the user's checkout.
   - First, whether gate 2 already merged the feature:
     - The branch is gone (`git rev-parse --verify --quiet refs/heads/feat/<feature>` prints nothing) and `main` holds the brief (`git cat-file -e main:.scratch/<feature>/brief.md` exits 0): gate 2 merged it and cleaned up. Its CI result may never have been reported: run *Gate 2* step 10, then end the run saying the feature is merged.
     - The branch exists, `main` holds it (`git merge-base --is-ancestor refs/heads/feat/<feature> main` exits 0), and the last `## Decision` entry of its brief (`git show feat/<feature>:.scratch/<feature>/brief.md`) says `merge: OK`: gate 2 merged it and stopped before the end. That `OK`, recorded and already in `main`, is the user's `OK` for the rest. Without entering or creating the worktree, go on at *Gate 2* step 6.
   - Already in that worktree: go on.
   - `git worktree list` shows it: switch the session into it with the EnterWorktree tool, passing `path`.
   - Otherwise the feature's tickets must be committed, because a new worktree only has what HEAD has: if `git status --porcelain -- .scratch/<feature>` prints anything, stop and say so. Create the worktree from the current HEAD with `git worktree add .claude/worktrees/<feature> -b feat/<feature>` (drop `-b` when the branch already exists), then switch into it with EnterWorktree, passing `path`.
   - In the worktree, uncommitted changes with no ticket `in-progress` are leftovers of a stopped run: stash them (`git stash push --include-untracked -m "ship: leftovers"`) and name the stash in your final report. With a ticket `in-progress` they are its work: leave them for the implementer.
4. **Check the plan usage** as *Plan usage* below describes, before anything builds: a full limit stops or waits here, not after a baseline run. This is the first ticket's check. Do it only when there is work to do: a ticket can be picked (see *Work the tickets*), or every ticket is done and there is no `brief.md` yet (see *Branch review*). Every ticket done and `brief.md` there: skip the checks and go to *Gate 2*, which builds nothing. With nothing to do, skip to *End of the run*.
5. **Check the environment** with `run-dod.ps1 -Check environment`.
6. **Check the baseline** with `run-dod.ps1 -Check commands`. The tickets were planned on a green build.

A failure in step 5 or 6, or exit 2 from either, is an **environment failure**: see below.

## Work the tickets

Read every ticket in `.scratch/<feature>/issues/` and pick the next one:

- a ticket whose status is `in-progress` (a run stopped in the middle of it) comes first;
- otherwise the lowest-numbered `ready-for-agent` ticket whose `Blocked by` tickets are all `done`;
- skip every other status. A ticket blocked by one that is not `done` waits.

For the picked ticket, first check the **plan usage** (below; the first ticket's check is the start's) and go on only when the check lets the ticket start. Then fix its **base**: set `**Status:** in-progress` and commit only that change, naming the usage just read: `ship: start <NN> (5h <n>%, week <n>%)`, with `no data` for a window the tool didn't give. A ticket that is already `in-progress` keeps its base: the latest `ship: start <NN>` commit on the branch (`git log -1 --grep "ship: start <NN>"`), or the current HEAD when there is none; commit nothing for it.

### Plan usage

The plan's usage limits are shared with everything else the user runs, and a ticket started near a limit stops halfway. So before each ticket, a resumed `in-progress` one included, and before the branch review and each of its fix rounds:

1. **Read the usage** with the app's usage tool (`get_usage`; load it with ToolSearch if it is deferred). Its plan windows give `percentUsed` and `resetsAt`: `5-hour limit` is the five-hour window, `Weekly · all models` the weekly one. When the plan part is not `ok`, ask once more. A window still missing counts as `0` in the check and as `no data` in the metrics; with neither window, skip the check and go on.
2. **Check it** with `run-dod.ps1 -Check quota -FiveHourPercent <five-hour percentUsed> -WeeklyPercent <weekly percentUsed>`. Exit 0: go on with the ticket or the branch step. Exit 2: an environment failure.
3. Exit 1 with `weekly` among the failed windows: a **weekly stop**. That window resets days later, so don't wait for it. If the last row of the metrics is already a weekly pause until the same reset (within an hour of it: the tool's `resetsAt` drifts by a moment between reads), the user started this run before the reset and is at the screen: end the run with no new row and no push. Otherwise add a pause row, commit only the metrics file (`ship: pause, weekly limit`), send a push notification `ship <feature>: weekly limit at <n>%, run /ship <feature> after <reset>`, and end the run.
4. Exit 1 with only `fiveHour` failed: a **five-hour wait**, without a push. If this run has already waited twice since it last finished a ticket, the branch review or a fix round, a reset didn't free the window (the usage tool is stale, or something else is spending the plan): don't wait a third time, it is an environment failure (`5-hour window`). Otherwise add a pause row and commit only the metrics file (`ship: pause, 5-hour limit`). Once that commit is in, run `wait-for-reset.ps1 -Until <resetsAt>`, passing the five-hour `resetsAt` exactly as the tool gave it, in the background (`run_in_background`, with the PowerShell tool), and do nothing else until it exits: the app holds the computer awake only while the session works, and the script holds it while the session waits. Exit 0: back to step 1. Any other exit is an environment failure (`wait-for-reset`).

Write a reset in a row or a push as local time, `date -d <resetsAt> "+%Y-%m-%d %H:%M"`. The check runs between tickets, never inside one, so the tracker already holds everything a later `/ship` resumes from.

### Rounds

A ticket gets at most **three rounds**. Its findings files, `.scratch/<feature>/findings/<NN>-round-<r>.md`, count the rounds already spent, so a resumed ticket goes on from the next one; one that already has three is **stuck**. Each round:

1. **Build.** Check the environment with `run-dod.ps1 -Check environment` first: the implementer must never work without it, and a pause or a stopped run may have taken it down since the last check. A failure is an environment failure. Then note HEAD and dispatch the implementer with the ticket path, the worktree path, the branch, the round number, the contract `docs/agents/implementer-contract.md`, and from round 2 the previous round's findings file. Round 3 runs the implementer on Opus (the Agent tool's `model: opus`): a ticket the Sonnet rounds couldn't land gets the stronger model before the loop gives up on it.
2. **Read its report**, once the implementer has really stopped: if it starts working again after reporting, the DoD gate has sent it back, so wait for its next report.
3. **Check the environment** with `run-dod.ps1 -Check environment`, before acting on the report. A failure here is an environment failure, not the ticket's fault. When its `started` names a check, the environment went down while the implementer worked, so its report says nothing about the ticket: dispatch the same round again (step 1), on the same model, with no findings file. That repeat doesn't count as a round and doesn't raise the model; a second repeat of the same round is an environment failure.
4. **Act on the report.**
   - `STATUS: stuck` in round 1 or 2: go straight to round 3 on Opus, passing the implementer's note as the findings. If the note says the ticket needs a decision from the user or a change to a protected path, no model can help: the ticket is **stuck**. `STATUS: stuck` in round 3, or a missing status line: the ticket is **stuck**.
   - Round 1 with no implementer commits after the base (commits whose subject doesn't start with `ship: `): the ticket is **stuck**.
   - Otherwise, stash anything uncommitted (`git stash push --include-untracked -m "ship: <NN> round <r> leftovers"`), so the verifier's DoD run and its diff check see the same committed state.
5. **Verify.** Dispatch the verifier with the worktree path, the ticket path and the base. It answers `VERDICT: pass` (with the criteria it confirmed), `VERDICT: fail` (with findings) or `VERDICT: error` (a script could not run: treat it as an environment failure).
6. **Review.** When the verifier passes, dispatch the spec reviewer with the ticket path, the base, the ticket's earlier findings files and the out-of-scope tickets already filed from it, so it doesn't report the same thing twice.
7. **Decide.**
   - No in-scope findings from either: the ticket is **done**.
   - In-scope findings: write them, numbered and with their source, to the round's findings file and commit it (`ship: <NN> findings round <r>`). After round 1 or 2, start the next round. After round 3 the ticket is **stuck**.
   - Out-of-scope findings never fail a round: each becomes a new ticket (below).

A verifier finding is never waived: a protected path changed, a suppression added, a weakened test, a project dropped from the solution or a red DoD always counts as in-scope, however good the rest is. The implementer may not touch a protected path, even to undo its own change, so put such a path back yourself before the next round: `git checkout <base> -- <path>`, then commit (`ship: <NN> restore <path>`). Say so in the findings file.

### Done

Set `**Status:** done`, tick the criteria the verifier confirmed, name any leftovers stash in the ticket, add the ticket's metrics row, and commit (`ship: <NN> done`).

### Stuck

Put the branch back to green without losing the work. Stash anything uncommitted (`git stash push --include-untracked -m "ship: <NN> stuck"`). Then undo the implementer's commits after the base, newest first, keeping your own `ship:` commits (findings files, restored paths, new tickets): `git revert --no-edit` on the commits that `git log --format=%h --invert-grep --grep "^ship: " <base>..HEAD` lists, in that order. Set `**Status:** needs-info` and add a **brief** under `## Comments`, short enough to read on a phone:

- **Tried:** the rounds spent and what each implementer did, in a line each.
- **Where it stands:** the last findings that were not resolved, the stash name, the reverted commits.
- **Needs:** the decision or information that would unblock it.

Add the ticket's metrics row, commit (`ship: <NN> needs-info`) and send a push notification (the harness's push-notification tool, if it has one): `ship <feature>: ticket <NN> needs info, see its Comments`. Tickets that depend on it now wait; the others go on.

### Out-of-scope findings

Each one becomes a ticket in the same `issues/` folder, with the next free number, `**Status:** needs-triage` and `**Blocked by:** None`. Its body is the finding and where it came from (ticket `<NN>`, round `<r>`). Skip a finding that repeats a ticket already filed from the same source ticket. Commit them (`ship: <NN> out-of-scope findings`). `needs-triage` is not `ready-for-agent`, so this run never picks them up: they wait for the user's triage and block nothing.

### Metrics

`.scratch/<feature>/metrics.md` records what each ticket cost, so the user decides about the plan from numbers. Create it with this header when it is missing:

```markdown
| Ticket | Result | Rounds | Models | Start | End | Minutes | 5h start | 5h end | Week start | Week end |
|---|---|---|---|---|---|---|---|---|---|---|
```

Rows go in the order things happened. Adding a row removes the total row, if there is one: only the end of a run writes it. Copy every time and number from the output of the command named for it, run on its own before the row is written; never write one from memory.

- **A ticket row** goes in just before the ticket's closing commit (`ship: <NN> done` or `ship: <NN> needs-info`), in that commit.
  - Result is `done` or `needs-info`; Rounds the rounds spent.
  - Models names the model of every role that ran, in the order they ran: `ship sonnet; impl sonnet, sonnet, opus; verifier haiku; reviewer sonnet`.
  - Start is the ticket's `ship: start <NN>` commit (`git log -1 --date=format-local:"%Y-%m-%d %H:%M" --format=%cd --grep "ship: start <NN>"`) and End is now (`date "+%Y-%m-%d %H:%M"`). Minutes is the difference of `date +%s` and `git log -1 --format=%ct --grep "ship: start <NN>"`, divided by 60. When the run resumed the ticket, add `, resumed` to the Result, because its minutes include the stop.
  - `5h start` and `Week start` come from that start commit's message, `5h end` and `Week end` from the usage read again now; `no data` where the tool gave none.
  - A ticket with no start commit takes its Start, Minutes and start usage from this run's check before it.
- **A branch row** goes in the brief's commit: `branch` as the ticket, `brief` or `brief, <n> open` as the result, the fix rounds spent as Rounds, and every role that ran in Models (`ship sonnet; standards opus; spec opus; impl sonnet; verifier haiku; re-review opus`). Its start is the commit that added `findings/branch-review.md`: Start is `git log --diff-filter=A -1 --date=format-local:"%Y-%m-%d %H:%M" --format=%cd -- .scratch/<feature>/findings/branch-review.md`, Minutes the difference of `date +%s` and the same command with `--format=%ct`, divided by 60, and the start usage comes from that commit's message (`--format=%s`). End and the end usage are now, as in a ticket row. When this run did not start the review, add `, resumed` to the Result.
- **A pause row** has `pause` as the ticket, `5-hour until <reset>` or `weekly until <reset>` as the result, now as Start (`date "+%Y-%m-%d %H:%M"`), and the usage that stopped the run in `5h start` and `Week start`; the other columns stay empty.
- **A total row** goes at the bottom when the run ends with nothing left that it can do (the brief is written, or no ticket can be picked while some wait on the user) and the table has none. It goes in the brief's commit, or else in its own (`ship: metrics total`).
  - `**Total**` as the ticket, `<n> done, <n> needs-info` as the result (counting ticket rows; a `done, resumed` row counts as `done`), and the sums of Rounds and Minutes over the ticket and branch rows.
  - In `5h end` and `Week end`, the summed usage of the ticket and branch rows (end minus start) as `+<n>`. Leave out of a sum a row with `no data`, or one whose end is below its start (the window reset during that ticket), and mark that sum with `*`.

Repeat until no ticket can be picked. Then, if every ticket is `done` (`needs-triage` tickets are new backlog and `wontfix` ones were dropped: neither is this feature's work), go on to the branch review. Otherwise the feature waits on the user: end the run.

## Branch review

Once every ticket is done, two reviewers on Opus look at the whole branch, the in-scope findings go back to the implementer for at most three fix rounds, and a brief tells the user whether to merge. The branch's base is `git merge-base main HEAD`. Its state is in `.scratch/<feature>/`, so a run goes on at the first of these that holds:

1. `brief.md` exists: the feature waits at gate 2. Go to *Gate 2*, with no push: the push went out with the brief. To have the branch reviewed again, the user deletes `brief.md` and `findings/branch-*.md` in a commit on the branch.
2. `findings/branch-review.md` is missing: **Start** below.
3. It holds no reviewer reports yet: **Review**.
4. There is no `findings/branch-round-<r>.md`: the review found nothing in-scope, write the **Brief**.
5. The highest round file ends with a `## Result`: write the **Brief**.
6. It has a `## Resolution` (the implementer finished round `r`): fix round `r` at **Verify**.
7. Otherwise: fix round `r` at **Build**.

A fix round's base is the commit that added its findings file: `git log --diff-filter=A -1 --format=%h -- .scratch/<feature>/findings/branch-round-<r>.md`. Round 1's base is the **fix base**: the verifier checks everything after it in every round, as it checks a ticket from its start. Check the plan usage (*Plan usage*) before the review and before each **Build**: Opus spends the plan fastest. A run that starts here has just checked it.

### Start and review

1. **Start.** Write `findings/branch-review.md` with one line, `Started (5h <n>%, week <n>%)`, the usage just read, and commit only that file: `ship: start branch review (5h <n>%, week <n>%)`.
2. **Review.** Dispatch both reviewers at once and wait for both:
   - `mattpocock-skills:standards-reviewer` with the base, the tickets folder, the findings folder, and `standardsSkill` from `dod.json` or that there is none;
   - `mattpocock-skills:spec-reviewer` with `model: opus`, the base, the spec path and the tickets folder instead of one ticket, and the out-of-scope tickets already filed.
3. **Record.** Add both reports as they came to `findings/branch-review.md`, under `## Standards` and `## Spec`. Write the in-scope findings of both, numbered and each with its axis, to `findings/branch-round-1.md`; with none, write no round file. Out-of-scope findings become tickets (*Out-of-scope findings*, from `branch review`). Commit (`ship: branch review findings`).
4. No in-scope findings: write the brief. Otherwise run fix round 1.

### Fix rounds

Fix round `r` resolves `findings/branch-round-<r>.md`. A ticket's round rules hold, with the findings file in place of the ticket: at most three rounds, the third on Opus, and a verifier finding is never waived.

1. **Build.** Dispatch the implementer as in a ticket's round (*Rounds*, steps 1 to 4: the environment checks around it and a repeat when it went down), with the spec path and the findings file instead of a ticket. `STATUS: stuck` in round 1 or 2 goes straight to round 3 on Opus as there: end the round file with `## Result` and `round 3`, write `findings/branch-round-3.md` with the findings left and the implementer's note, and commit both (`ship: branch findings round 3`). Stuck for good, as *Rounds* defines it, ends the fix rounds: the findings left are **open**.
2. **Verify.** Stash leftovers and dispatch the verifier as in a ticket's round, with the findings file instead of the ticket and the fix base.
3. **Re-review.** When the verifier passes, look at only this round's change, against the findings file, on Opus: `mattpocock-skills:standards-reviewer` when the file has Standards findings, `mattpocock-skills:spec-reviewer` with `model: opus` when it has Spec findings, both at once when it has both. Pass each the findings file and the round's base.
4. **Decide.** The round's result is the in-scope findings of the verifier and the re-review (a finding left unresolved, a fix that broke something, change beyond the findings), plus every finding of the round file that no re-review cleared, all of them when the re-review didn't run; each keeps its axis. Out-of-scope findings become tickets (`ship: branch out-of-scope findings`). Put a protected path the round changed back as *Rounds* says (`ship: branch restore <path>`).
   - None: end the round file with `## Result` and `resolved`, commit (`ship: branch round <r> resolved`) and write the brief.
   - After round 1 or 2: end the round file with `## Result` and `round <r+1>`, write the findings to `findings/branch-round-<r+1>.md`, commit both (`ship: branch findings round <r+1>`) and run the next round.
   - After round 3 they are **open**.

When the fix rounds end with open findings, the branch the user may merge must hold only verified work. If the last round's verifier passed it, keep the code. Otherwise (it failed, or never ran because the implementer was stuck), put the code back to where the last passed round ended, the base of the round after it, or to the fix base when no round passed, keeping the tracker as it is: `git restore --source=<that commit> --staged --worktree -- . ":(exclude).scratch/<feature>"`. In the same commit, end the last round file with `## Result`, `open`, the open findings and the commit the code went back to (`ship: branch round <r> open`). Then write the brief.

### Brief

Write `.scratch/<feature>/brief.md`, short enough to read on a phone. Take its facts from the tracker and from these commands, each run on its own:

- `run-dod.ps1 -Check commands`: copy, as printed, the lines of `passed` that report test results (passed, failed and skipped counts). A red run is the first open item.
- `get_session` with `"self"`: whether Remote Control is on.
- With `screenPaths` in `dod.json`: `git diff --name-only <base> HEAD -- <the globs>` lists the changed screens.

Its sections, in this order:

- **Ready:** one line: ready to merge, or what stands in the way (open findings, a red DoD).
- **Done:** each ticket with its title, rounds and result; the `needs-triage` tickets filed during the work.
- **Tests:** the summary lines.
- **Review:** each in-scope finding of the branch review with how its `## Resolution` resolved it, then the open ones.
- **Uncertain:** what deserves a look although nothing failed: the reviewers' notes, resumed tickets, stashes, pauses.
- **Screens:** only with `screenPaths`, the changed screen files.
- **Convention proposals:** the standards reviewer's proposals, numbered, each with its target file, its rule quoted exactly as the reviewer wrote it (in the target file's language, which may differ from the brief's), why, and the ticket it came from.
- **Remote Control:** on or off.
- **Gate 2:** the question *Gate 2* asks, with an example answer.

Add the branch row and the total row to the metrics, commit them with the brief (`ship: brief`), send a push notification `ship <feature>: ready for gate 2, see .scratch/<feature>/brief.md`, and go to *Gate 2*.

## Gate 2

The user decides in one message, in this session, from the computer or through Remote Control from the phone: whether to merge the branch into `main`, and separately about each convention proposal. This is the only question the run asks. The answers are kept in `brief.md`, under `## Decision`, so a later run knows what was settled. A run whose earlier gate 2 already merged the feature doesn't come here: *Start or resume*, step 3, sends it on. A session that started inside the feature's worktree can't leave it for the main checkout, where the merge happens: say that gate 2 runs from a `/ship` started in the main checkout, and end the run before asking anything.

The steps below decide from each command's exit code whether the next one runs, and they move `main` and the remote. So the rule of *Start or resume* holds here even in the main checkout, where the shell would accept a chain: one command per call, and read its exit code before the next. Never put the push or the cleanup behind `;` or `&&`.

1. **Ask.** Report as *End of the run* says, then end the turn with one question, in the brief's language: `OK` merges, a no keeps the branch, and each proposal no entry has decided yet takes `<n>: yes` or `<n>: no` (in a Polish brief `tak` and `nie`), with an example (`OK, 1 tak, 2 nie`). Send no push here: the brief's push already went, and a user who typed `/ship` is at the screen. When the last entry says `merge: OK` but `main` doesn't hold the branch (an earlier run stopped before the merge), say so in the question: an `OK` counts only in the session that merges, or once it is in `main`. The run goes on when the user answers.
2. **Read the answer.**
   - Only an unambiguous `OK` approves the merge; a clear no declines it. A question, a remark, an answer with no merge word or one that is neither, or anything you are not sure of decides nothing, the proposals included: record nothing, answer what was asked, ask again as in step 1, and end the turn.
   - Each open proposal: yes accepts it, no rejects it, an answer that doesn't mention it drops it. A dropped proposal stays undecided: a later gate 2 asks about it again.
3. **Record.** Add an entry under `## Decision` at the end of `brief.md` (add the heading the first time): the time (`date "+%Y-%m-%d %H:%M"`), `merge: OK` or `merge: no`, each proposal as accepted, rejected or dropped, and the answer verbatim. Write each accepted proposal's rule into its target file, exactly as the brief quotes it, where the proposal says, changing nothing else there: with putting a path back from the base, the only write to a protected path this skill makes. Commit the entry and the rules together, apart from any other change: `ship: gate 2, conventions <n>[, <n>]` when it accepts any, otherwise `ship: gate 2 decision`.
4. **No merge** ends the run: nothing went to `main`. Running `/ship <feature>` again asks again.
5. **Check `main`.** In the worktree, `git fetch origin`, then `git merge-base --is-ancestor origin/main HEAD`. Any exit but 0 from either means the fetch failed, the remote `main` moved past the branch's base, or there is none: stop. Never rebase: that is the user's.
6. **Leave the worktree** with the ExitWorktree tool, `action: "keep"` (it never removes a worktree that was entered by `path`); a run sent here by *Start or resume* never entered it and skips the tool. `git rev-parse --show-toplevel` must now print the main checkout, the first path in `git worktree list`; if it still prints the feature's worktree, the session started there: stop, and say that gate 2 merges from the main checkout.
7. **Fast-forward `main`** in the main checkout, by its branch (`git rev-parse --abbrev-ref HEAD`):
   - `main`, and `git status --porcelain --untracked-files=no` prints nothing: `git merge --ff-only feat/<feature>` (it refuses by itself to overwrite an untracked file);
   - `main` with changes: stop. They are the user's work in progress;
   - another branch: `git fetch . feat/<feature>:main`. It moves `main` only by a fast-forward, and refuses while `main` is checked out in another worktree.

   Any exit but 0: stop. Never `--update-head-ok`, a merge commit or a rebase. Run again after `main` already holds the branch, both commands change nothing and exit 0.
8. **Push.** First `git fetch origin`; when `git merge-base --is-ancestor refs/heads/feat/<feature> origin/main` then exits 0, the remote already holds the feature (an earlier run pushed it): go on at step 9. If not, `git rev-parse main` and `git rev-parse refs/heads/feat/<feature>` must print the same commit. A different one means `main` holds commits that are not the feature's, which no `OK` covered: stop. Then push with exactly `git push origin main`, never another refspec and never `--force`. Any exit but 0: stop. The local `main` holds the feature and the remote does not: say so, and leave the worktree and the branch.
9. **Clean up.** When `git worktree list` shows `.claude/worktrees/<feature>`, `git worktree remove .claude/worktrees/<feature>`. On Windows it can fail with `Permission denied` while a shell of this session still holds the folder: when `git worktree list` no longer shows it, git has dropped the worktree and only an empty folder is left, which the user deletes once the session is closed; say so. Only once `git worktree list` no longer shows the worktree, delete the branch at the tip the remote holds: take it from `git rev-parse refs/heads/feat/<feature>`, check that `git merge-base --is-ancestor <tip> origin/main` exits 0, then `git update-ref -d refs/heads/feat/<feature> <tip>`, which refuses if the branch moved meanwhile. (`git branch -d` would refuse a merged branch whenever the checkout is on another branch, because it compares with HEAD.) A failed removal keeps the branch, so a later run still finds both. A failure here doesn't stop the gate: report it and go on.
10. **Check CI.** Take the remote from `git remote get-url origin`. Not a `github.com` repository: the result is `no CI for this remote`. Otherwise open `https://github.com/<owner>/<repo>/actions?query=branch%3Amain` in the app's built-in browser and read it as text (load its tools with ToolSearch if they are deferred). The push's run is the one titled with the subject of the last commit pushed (`git log -1 --format=%s origin/main`). When a late resume finds `main` moved on since, no run matches and the result is `CI not checked`. While it is queued, running or not listed yet, wait (`Start-Sleep -Seconds 120` with the PowerShell tool, in the background) and read the page again, ten reads at most. No browser tool, a sign-in page instead of the runs, or no finished run after the tenth read: the result is `CI not checked`, with the link. A failed run: send a push notification `ship <feature>: merged, CI failed, see <link>`, and revert nothing.

A stop in steps 5 to 8 leaves the decision recorded and nothing to undo: at most the local `main` moved by a fast-forward. Name the step and the command's output in the final report. The user sets `main` or the remote right. The next `/ship <feature>` asks again while `main` doesn't hold the branch, and goes on at step 6 once it does.

## Environment failure

A check that won't come up (Docker, say), an unreadable DoD file, a red baseline, or a usage check that can't run or a five-hour wait that fails stops the whole feature: it is not the implementer's to fix. Send a push notification (the harness's push-notification tool, if it has one) with one line naming the failure (`ship <feature>: environment failure, docker`), then end with the `run-dod.ps1` output.

## End of the run

Report in a few lines: the tickets done, the tickets set to `needs-info` and why, the new `needs-triage` tickets, the tickets still waiting, the pauses, the branch, whether Remote Control was on, each push the tool reported as not sent with the reason it gave, and the brief when there is one; after a weekly stop, also when the limit resets. After gate 2, also the decision (merge, and each proposal accepted, rejected or dropped), the commit `main` was moved to, the push, the cleanup and the CI result with its link, or the step that stopped it with the command's output. Leave the worktree and the branch in place unless gate 2 removed them: merging is the user's call.

## Never

- Ask the user anything, or wait for an answer, except the question of *Gate 2*.
- Start a ticket while the usage check says stop, or wait out the weekly window in the session.
- Waive a finding, or fix a ticket yourself instead of sending it back to the implementer.
- Push, merge into another branch, or delete a branch or a worktree. The only exception is *Gate 2* after the user's `OK`, given in this session or recorded and already in `main` (*Start or resume*, step 3): the fast-forward of `main`, `git push origin main`, and, once that push succeeded, removing the feature's worktree and branch.
- Switch Remote Control on or off.
- Edit a path in `protectedPaths`, the hooks, `dod.json` or the implementer contract. Two writes are allowed there: putting a path back from the base, as the rounds describe, and writing a convention the user accepted at *Gate 2*, exactly as the brief quotes it.
- Build in the user's checkout instead of the worktree.
