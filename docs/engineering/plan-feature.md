## What it does

`plan-feature` takes a feature from the first question to [tickets](https://www.aihero.dev/ai-coding-dictionary/ticket) that [ship](https://aihero.dev/skills-ship) can build while you are away. In one [session](https://www.aihero.dev/ai-coding-dictionary/session) it runs the grilling of [grill-with-docs](https://aihero.dev/skills-grill-with-docs), the spec of [to-spec](https://aihero.dev/skills-to-spec) and the tickets of [to-tickets](https://aihero.dev/skills-to-tickets), each ticket with its seams and numbered acceptance criteria.

It asks for **one approval** at the end instead of three along the way. The seams and the ticket breakdown, which the standalone skills each check with you, wait for the package: the spec, the tickets, their seams and criteria, and the project's definition of done, shown together. Until you answer `OK`, the files sit uncommitted with `needs-triage`, so nothing can start building them.

## When to reach for it

You invoke this by typing `/plan-feature`, and the agent won't reach for it on its own.

| Your situation | Reach for |
| --- | --- |
| A feature you want `/ship` to build unattended, in a project with a definition of done | `/plan-feature` |
| You want to approve the seams and the breakdown one at a time, or your tracker isn't local markdown | [grill-with-docs](https://aihero.dev/skills-grill-with-docs), then [to-spec](https://aihero.dev/skills-to-spec) and [to-tickets](https://aihero.dev/skills-to-tickets) |
| An effort too big for one session | [wayfinder](https://aihero.dev/skills-wayfinder) |

## Prerequisites

- **A local markdown tracker** configured by [setup-matt-pocock-skills](https://aihero.dev/skills-setup-matt-pocock-skills): `/ship` reads the tickets from `.scratch/<feature>/`.
- **A definition of done** in `docs/agents/dod.json`, the file `/ship` runs against. Without it the skill stops before the first question.
- **Opus as the session's model.** The skill checks it first and asks you to switch when it is anything else.

## The package

The package is the whole plan on one screen, short enough to read on a phone: the spec's path, every ticket with what blocks it, its seams and its criteria with the test planned for each, and the definition of done with any change the feature needs. Only an unambiguous `OK` approves it. Anything else is a correction: the skill makes it, reopening the grilling where a decision changes, and shows the package again.

On `OK` the spec and tickets become `ready-for-agent` and go into one commit, `plan: <feature>`, together with the glossary, ADRs and definition-of-done changes the session made. Then the skill gives you the exact `/ship` command and how to run it.

## Common questions

**Why does it ask me to switch models instead of switching itself?**
A skill's own model setting lasts only for the turn it starts in, and a grilling has many turns. What holds is the session's model, and that is yours to set.

**Why should I be on `main` when I approve?**
`ship` builds the feature from the commit you are on, and merges it back into `main` only when `main` is behind the branch. A plan committed on another branch carries that branch's commits into the feature. The skill points this out and commits elsewhere only when you answer `OK here`.

**My session stopped halfway. Do I start over?**
No. A plan that was never committed stays on disk as a draft, and running `/plan-feature <feature>` again offers to go on from where it stopped: the grilling, the tickets or the package.

**Does it start `/ship` for me?**
No. It proposes the run, as a session you start where the app can offer one, and building begins only when you say so.

## It's working if

- The questions you answer are all design questions; nobody asks you about seams or the breakdown before the package.
- The package fits on your phone's screen, and you can approve it with one word.
- After `OK`, the history has one `plan: <feature>` commit holding the spec, the tickets and the docs, and `/ship <feature>` runs without asking anything.

## Where it fits

`plan-feature` is the planning step of the unattended chain, in place of the three planning steps of the main one:

```txt
plan-feature → ship → retro
```

It needs [grilling](https://aihero.dev/skills-grilling) and [domain-modeling](https://aihero.dev/skills-domain-modeling), and it follows the processes of `to-spec` and `to-tickets`, so a change to either changes it too. For the whole map, see [ask-matt](https://aihero.dev/skills-ask-matt).
