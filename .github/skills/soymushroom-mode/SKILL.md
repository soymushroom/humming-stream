---
name: soymushroom-mode
description: Use when the user invokes /soymushroom-mode or asks to plan, split, and track work in the soymushroom workflow, creating linked GitHub issues and the root todo.md.
---

# soymushroom-mode

Turn a requested plan into a durable, dependency-aware GitHub issue list and a matching local root `todo.md`. Use repository artifacts as context so a new session can continue without chat history.

## Plan and issue workflow

1. Read the applicable design source and repository instructions. For ScoreStream, VST host, MCP, or API work, read `docs/initial-plan.md` first. Inspect the current source, build manifests, existing issues and PRs, `.gitignore`, and `todo.md`; derive architecture and build commands from current files.
2. Derive a concise plan with deliverable outcomes, order, dependencies, and verification from those sources. Resolve ordinary details from repository evidence. Pause for user input only when an unresolved choice materially changes scope or architecture.
3. Inventory the issues and PRs in scope, including closed ones. Reuse matching issues rather than creating duplicates. Preserve existing details and historical outcomes when adding clarification.
4. Break work into verifiable issues. State each goal, scope and exclusions, observable acceptance criteria, direct dependencies and what each blocks, and the relevant design sections. Separate confirmed requirements from open decisions and hardware-dependent observations. Identify where a future decision or verification result must be recorded and which issue consumes it. When a prerequisite is missing, record it as a dependency.
5. In each PR, explain the change, rationale, related issues, actual verification and unrun checks. Link evidence instead of presenting untested behavior as verified. Keep merged PRs' historical results distinct from later findings.
6. Create or update GitHub records, then synchronize the intended checkout's root `todo.md` with open issue links and direct dependencies. Keep it local and ignored by Git. Confirm `/todo.md` is in `.gitignore`; add only that root-scoped rule if missing. Never stage or commit `todo.md`. It does not carry over to another clone or worktree.
7. Re-read every changed issue and PR, confirm their status and links, and compare the TODO dependencies with the issue bodies. Run `git check-ignore -v todo.md`. Use a separate context with only the TODO entry and normal repository access to interpret at least one representative issue: verify scope, dependency status, acceptance criteria and unresolved decisions against the sources. A skill discovery check alone is not a workflow test. Report untested behavior and incomplete writes explicitly.

## Delivery boundary

This workflow ends after issue creation and TODO synchronization. Create implementation branches or PRs only when the user includes delivery in the request. For a requested PR, branch from `main`, capture the decision rationale in `## Why`, and leave a PR comment with exact verification results and unrun checks. Merge and close issues only when the user explicitly requests that delivery step and required checks pass.
