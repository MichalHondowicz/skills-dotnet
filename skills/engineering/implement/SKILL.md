---
name: implement
description: "Implement a piece of work based on a spec or set of tickets."
disable-model-invocation: true
---

Implement the work described by the user in the spec or tickets.

If the user passes a ticket reference, fetch it from the issue tracker and state its title before starting. If the reference is ambiguous, ask.

Call the Skill tool with "tdd" where possible, at pre-agreed seams.

Run typechecking regularly, single test files regularly, and the full test suite once at the end. In .NET, typechecking is `dotnet build` (it also runs the analyzers, which a project may treat as errors) and a single test file is `dotnet test <test project> --filter "FullyQualifiedName~<TestClass>"`.

Once done, call the Skill tool with "mattpocock-skills:code-review" to review the work. That is this plugin's two-axis review; Claude Code's built-in "code-review" is a different skill that hunts bugs.

Commit your work to the current branch.
