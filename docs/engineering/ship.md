## What it does

`ship` runs a planned feature's [tickets](https://www.aihero.dev/ai-coding-dictionary/ticket) to the end while you are away. It moves the feature into its own git worktree, checks that the environment is up and the build is green, then hands the tickets one by one to an implementer [subagent](https://www.aihero.dev/ai-coding-dictionary/subagent) that builds each with [tdd](https://aihero.dev/skills-tdd) and commits it.

It asks you nothing until the feature is ready, and then exactly one question (see *Gate 2*). Every question an implementer could ask was settled when you approved the tickets: each one names its seams and the test that proves each acceptance criterion. What would otherwise be a question becomes a recorded outcome instead: a ticket that can't be finished in three rounds goes to `needs-info` with a brief on what was tried, and the tickets that don't depend on it go on.

## Rounds

Each ticket goes through up to three **rounds**. In each, the implementer builds; a **verifier** subagent on a cheap model runs the definition of done and a diff check (protected paths, analyzer suppressions, projects dropped from the solution, weakened tests) and confirms that every criterion names a test that was seen failing first; a **spec reviewer** then compares the change with the ticket. Their findings go back to the implementer as the next round's input, and the third round runs on a stronger model before the loop gives up. A problem a reviewer spots outside the ticket's own change becomes a new `needs-triage` ticket instead of holding this one back.

## Branch review and brief

When every ticket is done, two reviewers on the strongest model look at the whole branch once: one against the project's own coding standards (its standards skill, `ca-review` in a Clean Architecture template), one against the spec and all its tickets. Their in-scope findings go back to the implementer for at most three fix rounds, and each fix is checked again only against those findings, so the strong model reviews the whole branch once, not three times.

Then `ship` writes `brief.md` next to the tickets: whether the branch is ready, what was done, the test counts, how each review finding was resolved and which are still open, what deserves a look although nothing failed, the changed screens when the project names them, and the conventions the reviewers propose, each with its rule, reason and the ticket it came from. A notification points at the brief, so you decide about the merge from your phone. It reaches the phone only while the session has Remote Control on and mobile push is turned on in Claude Code's `/config`; `ship` reports whether it had Remote Control, and the reason for any push the tool didn't send.

## Gate 2

The brief ends with one question, and the session waits for your answer: the only question `ship` ever asks. You answer in one message, at the computer or through Remote Control on your phone: `OK` to merge or a no, plus a yes or no for each convention proposal (`OK, 1 yes, 2 no`). Only an unambiguous `OK` merges. A question or anything unclear gets an answer and the question again; a proposal you don't mention stays open for next time.

`ship` records your answer in the brief and writes each accepted convention into the file the reviewer named, word for word, in a commit of its own on the feature branch. On `OK` it then fast-forwards `main` without touching your checkout's files (it stops if `main` moved, if your checkout is on `main` with uncommitted work, or if the push fails), runs `git push origin main`, removes the feature's worktree and branch, and watches the CI run for that push in the app's browser for up to twenty minutes, unless `main` has no workflow in `.github/workflows` to run. A red CI sends a notification; nothing is reverted. In a project with no `origin` remote it merges into the local `main` only: no push and no CI check, and it still removes the worktree and branch.

## Usage limits

Before each ticket `ship` reads the plan's usage and compares it with the thresholds in the definition of done. It never starts a ticket at or above them, so a ticket doesn't stop halfway and some of the limit is left for you. When the five-hour window is full, it waits in the same session until the window resets, with a script holding the computer awake, and goes on by itself; you get no notification. When the weekly limit is full, it ends the run and sends one notification with the reset time, and you start `/ship` again after it.

Every ticket leaves a row in `metrics.md` next to the tickets: the usage of both windows at its start and end, the rounds, the model of each role, the time and the result. The run ends with a total, so how much of the limit a feature costs is a number, not a guess.

## When to reach for it

You invoke this by typing `/ship <feature>`, and the agent won't reach for it on its own.

| Your situation | Reach for |
| --- | --- |
| Approved tickets with seams, a project with a definition of done, and you want to walk away | `/ship` |
| You want to drive each ticket yourself, answering questions as they come | [implement](https://aihero.dev/skills-implement) |
| Tickets that should run at the same time, and you are there to watch | [implement-spec](https://aihero.dev/skills-implement-spec) |
| Tickets without seams or numbered criteria | [to-tickets](https://aihero.dev/skills-to-tickets) again first |

## Prerequisites

- **An issue tracker** configured by [setup-matt-pocock-skills](https://aihero.dev/skills-setup-matt-pocock-skills). `ship` reads the tickets there and writes their status back.
- **A definition of done** in `docs/agents/dod.json`: the environment checks (with an optional command that starts each one), the build and test commands, the protected paths, the name of the implementer role and the usage thresholds, and optionally the project's standards skill and the paths of its screens. The project also provides `.claude/hooks/run-dod.ps1`, which runs it and checks the usage, `.claude/hooks/wait-for-reset.ps1`, which waits out the five-hour window, `.claude/hooks/check-diff.ps1`, which the verifier runs over a ticket's commits, optionally `.claude/hooks/check-reboot.ps1`, which tells whether Windows is waiting to restart (`ship` then warns you with a push), optionally `.claude/hooks/notify.ps1`, which sends each push again to your own notification topic, and an implementer contract in `docs/agents/implementer-contract.md`.
- **Hooks that hold the implementer to the contract**: one refuses to let it report success while the definition of done is red, one refuses its writes to protected paths.

## The tracker is the state

`ship` keeps no progress of its own. A ticket it is working on is `in-progress`; each round's findings are a committed file; when a ticket finishes it becomes `done`, or `needs-info` with its brief, and each change is a commit on the feature branch. So starting and resuming are the same command: a run that stopped halfway (a closed laptop, a weekly limit) picks up at the `in-progress` ticket the next time you type `/ship <feature>`.

## Where it fits

`ship` is the build step of the main chain for when you won't be there, as the unattended alternative to running [implement](https://aihero.dev/skills-implement) once per ticket. [plan-feature](https://aihero.dev/skills-plan-feature) plans what it builds, running the grill, the spec and the tickets in one session and ending with one approval:

```txt
plan-feature → ship → retro
```

## Common questions

**Why does a stuck ticket undo its own commits?**
So the branch stays green for the tickets after it. The work is not lost: uncommitted changes go to a named stash, the commits stay in history behind a revert, and the ticket's note says where both are.

**Does it merge or push when it is done?**
Only after your `OK` at gate 2. Until then it leaves the feature branch and its worktree in place. After it, the only push is `git push origin main`, a fast-forward with no `--force`. The project's allowlist should name exactly that command, so a push of anything else still asks you.

**Why does it ask about each convention separately?**
A convention binds every later feature, so it changes only by your decision, not as a side effect of approving a merge. An accepted one lands in its own commit, so the history shows where the rule came from.

**Why doesn't it turn Remote Control on itself?**
The app lets a session switch it only when you ask, and refuses it to sessions running unattended. Turn on "Connect new sessions to Remote Control" in the app's settings once, and every `/ship` session starts connected.

**Why three rounds and not until it works?**
A ticket that two rounds and a stronger third couldn't land is rarely one more round away; it is usually missing a decision only you can make. Every extra round spends the usage limit the rest of the feature needs.

## It's working if

- You come back to a feature branch with one commit per ticket step and every ticket either `done` or `needs-info` with a brief you can act on from your phone.
- The findings files show the reviewers sending real problems back, and the next round fixing them.
- No permission prompt or question is waiting in the session.
- `metrics.md` has a row for every finished ticket, a row for every pause, a row for the branch review, and a total once nothing is left to do.
- A finished feature ends with `brief.md`, a notification pointing at it, and the branch review's findings either resolved or listed as open.
- After your `OK`, `main` on the remote holds the feature (the local `main`, in a project with no remote), the worktree and branch are gone, and the report names the CI result.
- Running `/ship <feature>` again on a merged feature only finishes what the merge left undone (the push, the cleanup, the CI check) and says so.
