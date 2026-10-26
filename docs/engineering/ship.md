## What it does

`ship` runs a planned feature's [tickets](https://www.aihero.dev/ai-coding-dictionary/ticket) to the end while you are away. It moves the feature into its own git worktree, checks that the environment is up and the build is green, then hands the tickets one by one to an implementer [subagent](https://www.aihero.dev/ai-coding-dictionary/subagent) that builds each with [tdd](https://aihero.dev/skills-tdd) and commits it.

It never asks you anything. Every question an implementer could ask was settled when you approved the tickets: each one names its seams and the test that proves each acceptance criterion. What would otherwise be a question becomes a recorded outcome instead: a ticket that can't be finished in three rounds goes to `needs-info` with a brief on what was tried, and the tickets that don't depend on it go on.

## Rounds

Each ticket goes through up to three **rounds**. In each, the implementer builds; a **verifier** subagent on a cheap model runs the definition of done and a diff check (protected paths, analyzer suppressions, projects dropped from the solution, weakened tests) and confirms that every criterion names a test that was seen failing first; a **spec reviewer** then compares the change with the ticket. Their findings go back to the implementer as the next round's input, and the third round runs on a stronger model before the loop gives up. A problem a reviewer spots outside the ticket's own change becomes a new `needs-triage` ticket instead of holding this one back.

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
- **A definition of done** in `docs/agents/dod.json`: the environment checks (with an optional command that starts each one), the build and test commands, the protected paths, the name of the implementer role and the usage thresholds. The project also provides `.claude/hooks/run-dod.ps1`, which runs it and checks the usage, `.claude/hooks/wait-for-reset.ps1`, which waits out the five-hour window, `.claude/hooks/check-diff.ps1`, which the verifier runs over a ticket's commits, and an implementer contract in `docs/agents/implementer-contract.md`.
- **Hooks that hold the implementer to the contract**: one refuses to let it report success while the definition of done is red, one refuses its writes to protected paths.

## The tracker is the state

`ship` keeps no progress of its own. A ticket it is working on is `in-progress`; each round's findings are a committed file; when a ticket finishes it becomes `done`, or `needs-info` with its brief, and each change is a commit on the feature branch. So starting and resuming are the same command: a run that stopped halfway (a closed laptop, a weekly limit) picks up at the `in-progress` ticket the next time you type `/ship <feature>`.

## Where it fits

`ship` is the build step of the main chain for when you won't be there, as the unattended alternative to running [implement](https://aihero.dev/skills-implement) once per ticket:

```txt
grill-with-docs → to-spec → to-tickets → ship → retro
```

## Common questions

**Why does a stuck ticket undo its own commits?**
So the branch stays green for the tickets after it. The work is not lost: uncommitted changes go to a named stash, the commits stay in history behind a revert, and the ticket's note says where both are.

**Does it merge or push when it is done?**
No. It leaves the feature branch and its worktree in place and reports what happened; merging is your decision.

**Why three rounds and not until it works?**
A ticket that two rounds and a stronger third couldn't land is rarely one more round away; it is usually missing a decision only you can make. Every extra round spends the usage limit the rest of the feature needs.

## It's working if

- You come back to a feature branch with one commit per ticket step and every ticket either `done` or `needs-info` with a brief you can act on from your phone.
- The findings files show the reviewers sending real problems back, and the next round fixing them.
- No permission prompt or question is waiting in the session.
- `metrics.md` has a row for every finished ticket, a row for every pause, and a total once nothing is left to pick.
- Running `/ship <feature>` again on a finished feature picks nothing and says so.
