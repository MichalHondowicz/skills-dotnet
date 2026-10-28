---
name: plan-feature
description: "Plan a feature in one session on Opus: grill, spec and tickets, approved as one package, ready for /ship."
disable-model-invocation: true
argument-hint: "[idea or feature-slug]"
---

Plan the feature `$ARGUMENTS` (or the idea the user describes next) from first question to tickets that `/ship` runs without asking anything. This skill runs the grill, `to-spec` and `to-tickets` in one session and replaces their separate approvals with **one decision about the whole package**: the spec, the tickets with their seams and criteria, and the definition of done. Until that decision nothing is committed and no ticket is `ready-for-agent`, so nothing can start building a plan the user hasn't approved.

Keep the whole plan in one context window: don't compact or clear before the package is approved, so the spec and the tickets build on the same grilling.

## 1. Check the project

- The tracker must be the local markdown one: `/ship` reads a feature's tickets from `.scratch/<feature>/`. If the issue tracker and triage labels weren't provided to you, tell the user to run `/setup-matt-pocock-skills` and stop. If the tracker is another one, say that `/ship` can't run its tickets, point the user at `/grill-with-docs`, `/to-spec` and `/to-tickets`, and stop.
- `docs/agents/dod.json`, the project's definition of done, must exist: `/ship` won't start without it, so a plan for it would wait on something else. Missing: say so and stop.
- This skill follows `../to-spec/SKILL.md` and `../to-tickets/SKILL.md` (paths relative to this skill's directory). If either is missing, tell the user to install `to-spec` and `to-tickets` beside it, and stop.
- Note which files are already changed (`git status --porcelain`). The plan's commit will take only the plan's own files, and the package names any of them that carried other changes first.

## 2. Run on Opus

Planning is where the decisions weigh most, so it runs on Opus. A skill's own model setting lasts only for the turn it is invoked in, and a grill has many turns, so what counts is the **session's** model. Check it now:

- where the harness offers a session tool (in the Claude desktop app, `get_session` with `"self"`; load it with ToolSearch if it is deferred), read its `model`;
- otherwise, the model your system prompt says you run on.

It is Opus when the model's id or name says `opus`, except `opusplan`, which plans on Opus but runs on Sonnet. Not Opus: tell the user to switch this session to Opus (`/model opus`, or the app's model picker) and answer here once they have, then end your turn. When they answer, check again and go on where you stopped. Check again, the same way, before writing the spec (step 5) and before showing the package again after corrections (step 7): a model switch or a fallback mid-plan would otherwise go unnoticed. The commit itself (step 8) needs no check.

## 3. Drafts of an earlier run

A plan is committed in one go, so a feature directory under `.scratch/` with no tracked file (`git ls-files -- .scratch/<feature>` prints nothing) is the **draft** of a `/plan-feature` that stopped before its commit. Find them in `git status --porcelain -- .scratch` (an untracked directory shows as `?? .scratch/<feature>/`). Changes inside a committed feature directory are not drafts: ignore them.

- `$ARGUMENTS` names a committed feature: it is planned already, so never overwrite it. Offer to repeat the run proposal (step 9), in case the earlier session stopped right after its commit, or to plan under a different slug.
- Drafts exist (the one `$ARGUMENTS` names, or any when it names none): ask whether to go on with one, naming each when there are several, or plan something new. To go on with one:
  1. Set the status line of its spec and tickets back to `needs-triage` where it says `ready-for-agent` (a run that stopped during its commit).
  2. List the uncommitted glossary, ADR and `dod.json` changes and ask which belong to this plan: those join its files, the rest stay out of its commit.
  3. Read what is there and go on at the first piece that is missing. No spec: the grill (step 4), starting from the draft's glossary and ADR changes as the record of the earlier grilling. A spec: the tickets (step 6), keeping the ticket files that exist and writing the rest, until every user story of the spec lands in some ticket's criteria. Everything there: check that coverage too, then the package (step 7).

## 4. Grill

Call the Skill tool twice, for "grilling" and "domain-modeling". Interview the user in rounds until the frontier is empty and they confirm you share an understanding, writing the glossary and ADRs inline as decisions crystallise.

Choose the feature's slug here (kebab-case, the directory name under `.scratch/`) and say it, so the user can change it. The slug must be new: if `.scratch/<slug>/` already exists, committed or as another draft, choose another.

## 5. Spec

Check the model again (step 2). Then read `../to-spec/SKILL.md` and follow its process, with these differences:

- Skip its check of the seams with the user. Sketch them as it says, but they are decided at the package (step 7).
- Write the spec to `.scratch/<feature>/spec.md`, with `**Status:** needs-triage` on the line under its title instead of the `ready-for-agent` label.
- Commit nothing.

`to-spec` and `to-tickets` are user-invoked, so read them as files; never call the Skill tool with either. The tool refuses user-invoked skills, and its refusal says not to repeat their steps any other way. Reading them here is not a way around that: the user typed `/plan-feature` to run them, and one source of their templates keeps the two paths from drifting apart.

## 6. Tickets

Read `../to-tickets/SKILL.md` and follow its process from the spec, with these differences:

- Skip its quiz of the breakdown (its step 4). The breakdown is decided at the package (step 7).
- Publish the tickets as local files with `**Status:** needs-triage`, not `ready-for-agent`. Where the project's tracker doc describes a ticket format for unattended work, follow it: every ticket names its seams from the project's test strategy and numbers its criteria `AC1…n`, each with its planned test.
- Commit nothing.

If the feature needs a change to the definition of done (a new command, `screenPaths` for a feature with screens, another protected path), make it in `docs/agents/dod.json` now. It is part of the package.

## 7. The package

Read the current branch (`git branch --show-current`; it prints nothing on a detached HEAD, which counts as not `main`). Then show the package in one message, short enough to read on a phone:

- the spec's path and the feature's slug;
- each ticket: number and title, `Blocked by`, its seams, and its criteria `AC1…n` with their planned tests;
- the definition of done: its commands, `protectedPaths` and the `quota` thresholds, and any change this plan makes to it;
- the files the commit will take: the spec, the tickets, and the plan's glossary, ADR and `dod.json` changes. If one of them carried other changes before the plan touched it (step 1), say so: the commit would take those too.

End with one question:

- on `main`: answer `OK` to approve the whole package, or say what to change;
- on any other branch: say that `/ship` builds the feature from the current HEAD and its gate 2 needs `origin/main` behind the branch, so committing the plan here would carry this branch's commits into the feature. Then ask: switch to `main` and answer `OK`, answer `OK here` to commit on this branch anyway, or say what to change.

Read the answer, and the branch again (`git branch --show-current`), since the user may have switched while you waited:

- **`OK`** on its own, now on `main`, or **`OK here`** on any branch: the package is approved. Go to step 8.
- **`OK`** on its own, still on another branch: not yet approved. Ask the branch question again.
- **Anything else that changes the plan**, `OK` with a condition included ("OK, but split ticket 2"): these are corrections. Make them in the files. A correction that reopens a design decision goes back through a grilling round for that branch first. Then check the model (step 2) and show the whole package again with its question.
- **A question**: answer it and ask the question again.
- **The user drops the plan** ("stop", "drop it"): leave everything as it is, uncommitted, and list what is left in the working tree: the spec and tickets (`needs-triage`) and the glossary, ADR and `dod.json` changes. Then end.

Only an unambiguous `OK` approves. When unsure what an answer means, it is not `OK`: ask.

## 8. Commit the plan

Run every command here as a plain command of its own, never joined with `;`, `&&` or a pipe: whether to go on depends on each one's result.

1. Check the working tree again (`git status --porcelain`). If a file of the package now carries changes that aren't the plan's, which the package didn't name, show the package again: the user approved something else.
2. Set the status line of the spec and of every ticket to `ready-for-agent`.
3. Add the package's files by name (`git add -- <path> ...`), never with `git add -A` or `.`, which would sweep in the user's own work.
4. Commit exactly those files, naming them: `git commit -m "plan: <feature>" -- <path> ...`. Naming the paths keeps out anything the user had already staged.
5. Check that `git status --porcelain -- .scratch/<feature>` prints nothing: `/ship` refuses a feature with uncommitted tickets.

If step 3, 4 or 5 fails (a hook, a missing identity, a file left over), stop and say why, naming the files. Before a commit exists, also put the draft back: unstage the package's files (`git reset -q -- <path> ...`) and set the status lines back to `needs-triage`, so the draft is as it was and running `/plan-feature <feature>` again goes on from the package. After the commit exists, leave it and don't propose the run until the leftover is explained.

## 9. Propose the run

Give the exact command that builds the feature, `/ship <feature>` (as a Claude Code plugin, `/mattpocock-skills:ship <feature>`), and how to run it: a new session, on Sonnet with effort `medium`, in auto mode, with Remote Control on so its gate 2 can reach the user's phone.

Where the harness has a tool that proposes a new session the user starts (in the Claude desktop app, `start_session` with `initiation: "own_initiative"`), also offer the run that way, with the model set to Sonnet where the tool takes one. Keep the exact command in your message either way: the user may start the session by hand. Never start `/ship` yourself: building starts only when the user says so.
