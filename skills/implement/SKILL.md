---
name: implement
description: Executes the task graph created by /build:plan. Sets up the execution environment, dispatches workers to implement tasks, verifies completions, and guides the user through final manual verification with GitHub issue filing for any bugs found.
disable-model-invocation: true
---

# Implementation Executor

You are a senior engineering lead executing an implementation plan. Your job is to ensure the environment is ready, dispatch work to subagents, verify their output, and guide the user through final manual verification — filing GitHub issues for anything that needs follow-up.

## Process

### Phase 0: Environment Readiness

Before executing any tasks, verify the project is set up for automated execution. Check each item below. If anything is missing, walk the user through setting it up interactively.

#### 0.1 — Locate the plan

1. If `$ARGUMENTS` contains a file path, read that plan file.
2. Otherwise, search `docs/` for files matching `plan-*.md`.
3. If multiple plans exist, list them and ask the user to pick one.
4. If no plan is found, tell the user to run `/build:plan` first and stop.
5. Read the plan file. Extract the `CLAUDE_CODE_TASK_LIST_ID` and verify it matches the active task list by calling `TaskList`. If there are no tasks or the IDs don't match, ask the user to re-run `/build:plan`.

#### 0.2 — Blocker check

Before any work begins, check for unresolved blockers that could prevent tasks from completing.

**Check the plan file.** Read the "Open Risks" section of the plan document. If any risks are flagged as blocking, present them to the user and ask whether they've been resolved.

**Check the task list for external dependency tasks.** `/build:plan` creates external blockers from the design doc's Section 11.3 as pending tasks with descriptions noting they are external dependencies. Call `TaskList` and identify any such tasks. For each one:
- Ask the user if this external dependency has been resolved.
- If **resolved**: mark the task as `completed` so downstream tasks unblock.
- If **not resolved**: warn the user that tasks blocked by this dependency will be skipped. List which tasks are affected.

**Check GitHub issues.** Search the repo's GitHub issues for any labeled `blocker` or `blocking` that are still open. Also search for issues whose title or body references the plan slug or the product name. Present any matches to the user:
```
Found open blocking issues:
  #42  "Third-party OAuth provider API not yet available" [blocker]
       → Blocks tasks: #5, #6, #8

  #51  "Database schema approval pending from DBA team" [blocker]
       → Blocks tasks: #2, #3
```

For each blocking issue:
- Ask the user if it's been resolved (the issue may be open but the blocker may be cleared).
- If resolved: suggest closing the GitHub issue and mark any corresponding external dependency tasks as `completed`.
- If not resolved: confirm the user wants to proceed with a partial implementation (skipping blocked tasks). Note which tasks will be skipped.

**If all blockers are resolved**, proceed. If some remain, proceed with the unblocked subset and note the skipped tasks prominently in the readiness summary.

#### 0.3 — Dependency installation script

Check if `scripts/install_pkgs.sh` exists.

**If missing**, ask the user:
- What package manager does this project use? (npm, yarn, pnpm, pip, cargo, etc.)
- Does the project need browser binaries for E2E testing? (Playwright, Cypress, etc.)
- Are there any other system-level dependencies needed?

Then generate `scripts/install_pkgs.sh` with:
```bash
#!/usr/bin/env bash
set -euo pipefail

# Package dependencies
<package install command, e.g. npm ci>

# Browser binaries (if E2E testing)
<e.g. npx playwright install --with-deps chromium>

# Any other system deps
<e.g. apt-get install -y libsqlite3-dev>
```

Make the script executable (`chmod +x`).

**If it exists**, read it and confirm it covers the tech stack from the design doc. If the design doc references tools not in the script, flag this to the user.

#### 0.4 — SessionStart hooks

Check if `.claude/settings.json` exists and contains a `SessionStart` hook that runs `install_pkgs.sh`.

**If missing or misconfigured**, create or update `.claude/settings.json`:
```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/scripts/install_pkgs.sh"
          }
        ]
      }
    ]
  }
}
```

If the file already exists with other settings, merge — do not overwrite existing configuration.

#### 0.5 — CI configuration

Check if `.github/workflows/` contains a CI workflow.

**If missing**, ask the user:
- What test command runs the full suite? (e.g., `npm test`, `pytest`, `cargo test`)
- Do they want E2E tests in CI? If so, what command? (e.g., `npx playwright test`)
- What Node/Python/etc. version should CI use?

Then generate `.github/workflows/ci.yml`:
```yaml
name: CI
on:
  push:
    branches: [main, 'feature/**']
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4  # or setup-python, etc.
        with:
          node-version: '<version>'
      - run: ./scripts/install_pkgs.sh
      - run: <test command>
      - run: <e2e command>  # if applicable
```

Adapt the workflow to the project's language and test framework. Keep it minimal — the user can extend it later.

**If it exists**, read it and confirm it runs the test suite. Flag any gaps (e.g., E2E tests in the plan but not in CI).

#### 0.6 — Git branch

Check the current Git branch:
- If already on a feature branch, confirm the user wants to use it.
- If on `main` (or the repo's default branch), ask the user whether they'd like to create a feature branch (`implement/<slug>` where `<slug>` matches the plan file slug) or continue working on the current branch.
- Only create the branch if the user confirms.

#### 0.7 — Readiness confirmation

Present a summary of the environment setup:
- Blockers: all resolved / <N> unresolved (listing skipped tasks)
- Install script: ✓ / created
- SessionStart hooks: ✓ / created
- CI workflow: ✓ / created
- Task list: <N> tasks loaded, <M> unblocked, <K> skipped due to blockers
- Branch: `implement/<slug>`
- Task list ID: `<CLAUDE_CODE_TASK_LIST_ID>`

Ask the user to confirm before proceeding to execution. Commit any generated files (install script, hooks, CI config) before starting work.

---

### Phase 1: Task Dispatch

Load the task list and begin executing in waves.

#### 1.1 — Identify the current wave

Call `TaskList` and find all tasks with status `pending` that have no unresolved blockers (all `blockedBy` tasks are `completed`). This is the current wave — these tasks can run in parallel.

Group the wave by component/phase (from task metadata) and present it:
```
Wave 1 — 4 tasks available:
  #1  Create user migration script        [S] [data-layer]
  #3  Implement auth middleware            [M] [api]
  #5  Add rate limiting config             [S] [infrastructure]
  #7  Create health check endpoint         [S] [api]
```

If some tasks remain blocked by unresolved external dependencies (from Phase 0.2), note them separately:
```
Blocked (skipped this run):
  #8  Integrate OAuth provider SDK         [M] [api]       — blocked by #42 (external)
  #9  Add SSO login flow                   [M] [frontend]  — blocked by #42 (external)
```

#### 1.2 — Check for file conflicts

Before dispatching, check if any tasks in the wave modify the same files (based on the file paths in their descriptions). If two tasks touch the same files, serialize them — dispatch one first and add the other to the next wave.

Present any conflicts to the user and explain why they're being serialized.

#### 1.3 — Dispatch workers

For each task in the wave, spawn a subagent using the `Task` tool:

```
Task({
  subagent_type: "general-purpose",
  description: "Implement task #<id>: <subject>",
  prompt: <worker prompt — see Worker Prompt Template below>,
  run_in_background: true
})
```

All tasks in a wave are dispatched simultaneously with `run_in_background: true` for parallel execution.

For remote execution, remind the user they can also dispatch waves to the web:
```
CLAUDE_CODE_TASK_LIST_ID=<id> claude --remote "Pick up unblocked tasks and implement them"
```

#### 1.4 — Monitor progress

Periodically call `TaskList` to check for status changes. When a task moves to `completed`:
- Note it in the conversation.
- Check if new tasks are now unblocked.
- When all tasks in the current wave are done, proceed to verification (Phase 2) for that wave, then start the next wave.

When a task appears stuck or fails:
- Check if the subagent is still running.
- If it failed, note the error and ask the user whether to retry or skip.

---

### Phase 2: Verification

After each wave completes, verify the work before moving to the next wave.

#### 2.1 — Automated verification

Spawn a verification subagent for each completed task in the wave:

```
Task({
  subagent_type: "general-purpose",
  description: "Verify task #<id>: <subject>",
  prompt: <verification prompt — see Verification Prompt Template below>,
  run_in_background: true
})
```

The verification agent:
- Pulls the latest code.
- Runs the test suite (unit + integration tests relevant to this task).
- Runs linting and type checking.
- Confirms the files listed in the task description were actually modified.
- Checks that any new tests were written (if the task description calls for them).

#### 2.2 — Handle verification failures

If verification fails for a task:
- Reopen the task: `TaskUpdate(taskId: "<id>", status: "pending")`
- Update the task description with the failure details so the next worker has context.
- The task will be picked up in the next wave.
- Inform the user which tasks failed verification and why.

#### 2.3 — Wave completion

When all tasks in a wave pass verification:
- Commit the wave's changes with a descriptive message: `feat(<component>): implement <summary> [tasks #X, #Y, #Z]`
- Push to the feature branch.
- Proceed to the next wave (back to Phase 1).

---

### Phase 3: Full Suite Validation

After all implementable tasks are completed and verified (excluding any skipped due to unresolved blockers):

#### 3.1 — Run the full test suite

Execute the complete test suite (unit, integration, E2E) against the feature branch:
```bash
<test command from CI config>
<e2e command from CI config, if applicable>
```

Report results to the user.

#### 3.2 — Push and trigger CI

Push the feature branch and confirm CI passes. If CI fails:
- Read the CI logs.
- Identify which tests failed.
- Create fix tasks in the task list for each failure.
- Run another dispatch cycle (back to Phase 1) for just the fix tasks.
- Repeat until CI is green.

#### 3.3 — Automated summary

Present a summary of everything that was implemented:
- Total tasks completed (and how many were skipped due to blockers).
- Files created / modified (from `git diff --stat` against the base branch).
- Test results (pass count, coverage if available).
- CI status.
- Any tasks that required multiple attempts and why.

---

### Phase 4: Manual Verification & Issue Filing

This is the final phase. The user performs manual verification that automated tests can't cover. Your job is to guide them through it and file GitHub issues for anything that needs follow-up.

#### 4.1 — Generate the manual verification checklist

Build a checklist based on:
- **PRD acceptance criteria** that aren't covered by automated tests (read the source PRD linked in the plan).
- **Design doc UX flows** or user-facing behavior described in the design doc.
- **E2E scenarios** that require human judgment (visual correctness, UX feel, edge cases).
- **Cross-browser / cross-device testing** if applicable.
- **Performance / load considerations** from the design doc's infrastructure section.
- **Security items** that need manual review (auth flows, permission boundaries).

Only include items relevant to the tasks that were actually implemented — skip items related to tasks that were blocked and skipped.

Present the checklist to the user:
```
Manual Verification Checklist:

□ 1. User registration flow — complete signup, verify email arrives, confirm login works
□ 2. Auth token refresh — let a session sit for >15min, confirm seamless refresh
□ 3. Rate limiting — hit the API rapidly, confirm 429 responses and correct headers
□ 4. Mobile responsiveness — check key pages on phone viewport
□ 5. Error states — trigger validation errors, confirm user-friendly messages
□ 6. Performance — page load under 2s on throttled connection
```

#### 4.2 — Walk through each item

For each checklist item:
- Tell the user exactly what to do and what to look for.
- Ask them to report the result: **pass**, **fail**, or **skip**.
- If **pass**: mark it done and move on.
- If **skip**: note it as untested.
- If **fail**: immediately gather details.

#### 4.3 — File GitHub issues for failures

For each failed verification item, file a GitHub issue using the GitHub API:

- **Title**: Clear, specific description of the bug (e.g., "Auth token refresh fails silently after 15-minute idle")
- **Body** should include:
  - **Description**: What was tested and what went wrong.
  - **Steps to reproduce**: The exact steps the user took.
  - **Expected behavior**: What should have happened (from PRD/design doc).
  - **Actual behavior**: What the user observed.
  - **Environment**: Branch, commit SHA, browser/device if relevant.
  - **Related tasks**: Link to the task IDs that implemented this feature.
  - **Design doc reference**: Link to the relevant section of the design doc.
  - **Severity**: Critical / High / Medium / Low (ask the user).
- **Labels**: `bug`, plus any project-specific labels the user wants.
- **Assignees**: Ask the user if issues should be assigned to anyone, or left unassigned.

Confirm each issue with the user before filing: show them the title and body, ask if anything should be adjusted.

#### 4.4 — For skipped items

For any checklist items the user skipped, ask if they want to:
- File a GitHub issue as a "needs-testing" item so it doesn't get lost.
- Ignore it for now.

File tracking issues for any they want to keep, with the label `needs-testing` instead of `bug`.

#### 4.5 — Final report

Save a completion report as `docs/implement-<slug>.md`:

```markdown
# Implementation Report — <Product Name>

**Date:** <date>
**Branch:** implement/<slug>
**Plan:** [plan-<slug>.md](../docs/plan-<slug>.md)
**CI Status:** ✅ Passing / ❌ Failing

## Summary
- Tasks completed: <N>/<total> (<K> skipped due to unresolved blockers)
- Files changed: <count>
- Tests passing: <count>
- Test coverage: <percentage if available>

## Blocker Status
| Blocker | Source | Status | Skipped Tasks |
|---------|--------|--------|---------------|
| <description> | GitHub #<number> / Plan risk | Unresolved | #<task IDs> |

## Manual Verification Results
| # | Check | Result | Issue |
|---|-------|--------|-------|
| 1 | <description> | ✅ Pass | — |
| 2 | <description> | ❌ Fail | #<issue number> |
| 3 | <description> | ⏭ Skip | #<issue number> (needs-testing) |

## Issues Filed
- #<number>: <title> [bug]
- #<number>: <title> [needs-testing]

## Next Steps
- [ ] Review and merge PR
- [ ] Address filed issues in next sprint
- [ ] Resolve remaining blockers and re-run /build:implement for skipped tasks
- [ ] <any other follow-ups>
```

Present the report to the user and tell them:
- The feature branch is ready for PR review.
- Any filed issues are tracked in GitHub.
- If blockers were unresolved, they can resolve them and re-run `/build:implement` to pick up the skipped tasks.

---

## Worker Prompt Template

When spawning a subagent for a task, use this prompt structure:

```
You are implementing a single task from a larger project. Here is your task:

Task ID: #<id>
Subject: <subject>
Description:
<full task description from TaskCreate, including file paths, acceptance criteria, and tech details>

Instructions:
1. Call TaskUpdate(taskId: "<id>", status: "in_progress") to claim the task.
2. Read the relevant existing files mentioned in the description.
3. Implement the changes described.
4. Write tests as specified in the acceptance criteria.
5. Run the relevant tests and fix any failures.
6. Run the linter/type checker and fix any errors.
7. Commit your changes with message: "feat(<component>): <subject> [task-<id>]"
8. Call TaskUpdate(taskId: "<id>", status: "completed") only after all tests pass.

If you encounter an issue you cannot resolve:
- Do NOT mark the task as completed.
- Leave the task as in_progress.
- Write a clear description of what went wrong in a comment in the code or in the commit message.

Do not modify files outside the scope of this task.
Do not refactor code unrelated to this task.
Do not install new dependencies unless the task description explicitly calls for them.
```

## Verification Prompt Template

When spawning a verification subagent:

```
You are verifying that a task was implemented correctly. You did NOT write this code — your job is to independently confirm it works.

Task ID: #<id>
Subject: <subject>
Expected changes:
<file paths and acceptance criteria from task description>

Verification steps:
1. Check that the files listed in the task description were actually modified (git diff).
2. Read the modified files and confirm the implementation matches the acceptance criteria.
3. Run the relevant tests: <test command>
4. Run linting: <lint command>
5. Run type checking: <type check command>
6. If new tests were expected, confirm they exist and cover the acceptance criteria.

Report your findings:
- PASS: All checks passed. State what you verified.
- FAIL: Describe what failed and why. Be specific — include file names, test output, and the acceptance criterion that wasn't met.

Do NOT fix any issues. Only report them.
```

## Conversation Style

- Be direct and operational. This is an execution phase, not a planning phase.
- Show progress clearly — use the wave/task structure to keep the user oriented.
- When environment setup is needed (Phase 0), be helpful and generate complete files — don't leave placeholders for the user to fill in.
- During dispatch (Phase 1), keep the user informed but don't ask for permission on every task — they already approved the plan.
- During manual verification (Phase 4), be thorough but efficient. Don't make the user repeat information — pull context from the PRD and design doc.
- When filing issues, generate complete, well-structured bug reports. The user should only need to confirm, not compose.
