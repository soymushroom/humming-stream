---
name: soymushroom-mode
description: Use when the user invokes /soymushroom-mode or asks to plan, split, and track work in the soymushroom workflow, creating linked GitHub issues and the root todo.md.
---

# soymushroom-mode

Turn a requested plan into a durable, dependency-aware GitHub issue list and a matching local root `todo.md`. Use repository artifacts as context so a new session can continue without chat history.

## Plan and issue workflow

1. Read the applicable design source and repository instructions. For ScoreStream, VST host, MCP, or API work, read `docs/initial-plan.md` first. Inspect the current source, build manifests, existing issues and PRs, `.gitignore`, and `todo.md`; derive architecture and build commands from current files.
2. Derive a concise plan with deliverable outcomes, order, dependencies, and verification from those sources. Resolve ordinary details from repository evidence. Pause for user input only when an unresolved choice materially changes scope or architecture.
3. Compare the plan with open and recently closed issues. Reuse or update matching issues instead of creating duplicates. Preserve existing issue detail when editing.
4. Break work into independently deliverable issues. Each issue states its goal, scope and exclusions, observable acceptance criteria, actual blocking dependencies, and links to the design source. Keep each issue small enough to verify. Create a prerequisite issue when required project scaffolding is absent, and link blocked implementation issues to it rather than calling them dependency-free.
5. Make issue bodies self-contained. Record decisions and rationale in the relevant issue or PR so a new session can proceed from repository and GitHub context alone. Match the language and terminology used by nearby issues.
6. Create or update GitHub issues, then synchronize the root `todo.md` with issue links and direct dependencies. Keep it local and ignored by Git. Confirm `/todo.md` is in `.gitignore`; add only that root-scoped rule if missing. Never stage or commit `todo.md`.
7. Verify by rereading changed issues, checking that dependency links match `todo.md`, and running `git check-ignore -v todo.md`. Report any GitHub write or verification that could not be completed.

## Delivery boundary

This workflow ends after issue creation and TODO synchronization. Create implementation branches or PRs only when the user includes delivery in the request. For a requested PR, branch from `main`, capture the decision rationale in `## Why`, and leave a PR comment with exact verification results and unrun checks. Merge and close issues only when the user explicitly requests that delivery step and required checks pass.
