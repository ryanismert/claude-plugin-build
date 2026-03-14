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

Load the task list and begin executing in waves using a TDD two-phase approach.

#### 1.1 — Identify the current wave

Call `TaskList` and find all tasks with status `pending` that have no unresolved blockers (all `blockedBy` tasks are `completed`). This is the current wave — these tasks can run in parallel.

Group the wave by component/phase (from task metadata) and present it:
```
Wave 1 — 4 tasks available:
  #1  Create user migration script        [S] [data-layer]   → backend worker
  #3  Implement auth middleware            [M] [api]          → backend worker
  #5  Add rate limiting config             [S] [infrastructure] → infra worker
  #7  Create health check endpoint         [S] [api]          → backend worker
```

If some tasks remain blocked by unresolved external dependencies (from Phase 0.2), note them separately:
```
Blocked (skipped this run):
  #8  Integrate OAuth provider SDK         [M] [api]       — blocked by #42 (external)
  #9  Add SSO login flow                   [M] [frontend]  — blocked by #42 (external)
```

#### 1.2 — Select worker profiles

For each task in the wave, select a worker profile by reading `references/worker-profiles.md` and matching the task's `component` metadata tag against the profile selection table. If the task description contains file paths, also check those against the file pattern column. If no match is found, use the `general` profile.

Show the profile assignments in the wave summary (as in 1.1 above).

#### 1.3 — Check for file conflicts

Before dispatching, check if any tasks in the wave modify the same files (based on the file paths in their descriptions). If two tasks touch the same files, serialize them — dispatch one first and add the other to the next wave.

Present any conflicts to the user and explain why they're being serialized.

#### 1.4 — Phase A: Test writing (RED)

The test writer prompt is defined inline here (not in a reference file) because test writing is a single discipline that doesn't vary by task type — unlike implementation, which uses specialized worker profiles from `references/worker-profiles.md`.

For each task in the wave, spawn a test-writer subagent:

```
Agent({
  subagent_type: "general-purpose",
  model: "sonnet",
  description: "Write failing tests for task #<id>: <subject>",
  prompt: <see Test Writer Prompt below>,
  run_in_background: true
})
```

All test-writer agents in a wave are dispatched simultaneously.

**Test Writer Prompt:**

````
You are a test architect writing tests for a feature that has NOT been implemented yet. Your tests MUST fail when you run them — that is the whole point. You are writing the specification, not the implementation.

Task ID: #<id>
Subject: <subject>
Description:
<full task description, including acceptance criteria>

Test command: <test command from CI config>

Instructions:
1. Read the task description carefully. Each acceptance criterion becomes one or more test cases.
2. Read existing test files in the project to understand the testing patterns, frameworks, and conventions used.
3. Write test cases that:
   - Cover every acceptance criterion in the task description.
   - Test edge cases mentioned in the description.
   - Follow the project's existing test patterns and naming conventions.
   - Are specific and descriptive in their test names (the name should describe the expected behavior).
4. Run the tests. They MUST fail. If any test passes, you have either:
   - Written a trivially true assertion (fix the test), or
   - Discovered the feature already exists (note this in your report).
5. Commit the failing tests with message: "test(<component>): add failing tests for <subject> [task-<id>]"
6. Report what you wrote: list each test case, what it tests, and confirm it fails.

IMPORTANT:
- Do NOT write any implementation code. Only tests.
- Do NOT write tests that pass. Every test must fail with a meaningful error (not found, not defined, assertion failed, etc.).
- Do NOT stub or mock the feature you are testing — test the real interface as described in the acceptance criteria.
````

Wait for all test-writer agents to complete. If any test-writer agent reports issues (couldn't determine test patterns, ambiguous acceptance criteria), surface these to the user before proceeding.

#### 1.5 — Phase B: Implementation (GREEN)

After all tests are written and confirmed failing, spawn implementer subagents using the selected worker profile:

```
Agent({
  subagent_type: "general-purpose",
  model: "sonnet",
  description: "Implement task #<id>: <subject>",
  prompt: <worker prompt from selected profile in references/worker-profiles.md>,
  run_in_background: true
})
```

Append this additional instruction block to the selected worker profile prompt:

````
IMPORTANT — TDD context:
Failing tests have already been written for this task. Your job is to make them pass.
- Run the tests FIRST to see what is expected.
- Write the minimal code to make each test pass.
- Do NOT modify the existing test files unless a test has a genuine bug (wrong import path, typo in fixture name). If you believe a test is wrong, note it in your commit message but still try to make it pass.
- After all tests pass, run the full test suite to check for regressions.
- Only then commit and mark the task complete.
````

All implementer agents in a wave are dispatched simultaneously with `run_in_background: true`.

#### 1.6 — Monitor progress

Periodically call `TaskList` to check for status changes. When a task moves to `completed`:
- Note it in the conversation.
- Check if new tasks are now unblocked.
- When all tasks in the current wave are done, proceed to verification (Phase 2) for that wave, then start the next wave.

When a task appears stuck or fails:
- Check if the subagent is still running.
- If it failed, note the error and ask the user whether to retry or skip.

For remote execution, remind the user they can also dispatch waves to the web:
```
CLAUDE_CODE_TASK_LIST_ID=<id> claude --remote "Pick up unblocked tasks and implement them"
```

---

### Phase 2: Verification

After each wave completes, verify the work with parallel multi-lens review before moving to the next wave.

#### 2.1 — Dispatch parallel reviewers

Read `references/reviewer-profiles.md` for the three reviewer prompt templates. For each completed task in the wave, spawn all three reviewers in parallel:

```
Agent({
  subagent_type: "general-purpose",
  model: "sonnet",
  description: "Correctness review: task #<id>",
  prompt: <correctness reviewer prompt from references/reviewer-profiles.md>,
  run_in_background: true
})

Agent({
  subagent_type: "general-purpose",
  model: "opus",
  description: "Security review: task #<id>",
  prompt: <security reviewer prompt from references/reviewer-profiles.md>,
  run_in_background: true
})

Agent({
  subagent_type: "general-purpose",
  model: "sonnet",
  description: "Quality review: task #<id>",
  prompt: <quality reviewer prompt from references/reviewer-profiles.md>,
  run_in_background: true
})
```

This means a wave of 4 tasks spawns 12 reviewer agents (3 per task), all running in parallel.

#### 2.2 — Visual verification (frontend tasks only)

For tasks that matched the `frontend` worker profile, spawn an additional visual verification agent **after** the three standard reviewers complete (it needs the implementation to be confirmed working first):

```
Agent({
  subagent_type: "general-purpose",
  model: "sonnet",
  description: "Visual verification: task #<id>",
  prompt: <see Visual Verification Prompt below>,
  run_in_background: true
})
```

**Visual Verification Prompt:**

````
You are a visual QA engineer verifying that a frontend task renders correctly. You did NOT write this code.

Task ID: #<id>
Subject: <subject>
Expected visual behavior:
<acceptance criteria related to UI appearance, layout, responsiveness from task description>

Instructions:
1. Start the dev server if not already running (check the project's package.json for the dev/start script).
2. Navigate to the relevant page or component using the browser tools available to you.
3. Take a snapshot (accessibility tree) to verify the DOM structure is correct.
4. Take a screenshot to verify the visual appearance.
5. If the task mentions responsive behavior or mobile support, resize the viewport to mobile dimensions (375x667) and take another screenshot.
6. Check the browser console for JavaScript errors or warnings.
7. Evaluate what you see against the acceptance criteria.

Report your findings:
- PASS: Visual output matches acceptance criteria. Describe what you verified.
- FAIL: Describe what looks wrong. Include:
  - What you expected to see (from acceptance criteria)
  - What you actually see (describe the visual issue)
  - Whether there are console errors
  - Screenshots taken (reference them in your report)

Do NOT fix any issues. Only report them.

NOTE: If no browser automation tools are available (no Chrome DevTools MCP, no Playwright), report SKIP with the reason. Visual verification is optional — the task can still pass based on the other three reviewers.
````

Visual verification is **opt-in and gracefully degraded**:
- Only runs for frontend-tagged tasks.
- Only runs if browser tools (Chrome DevTools MCP, Playwright) are available.
- If unavailable, the visual reviewer reports SKIP and the task is judged on the other three reviewers alone.
- Never blocks a task from passing when browser tools aren't configured.

#### 2.3 — Handle verification failures

Aggregate results from all reviewers for each task. A task's verification status is:

- **PASS**: All required reviewers (correctness + security + quality) passed. Visual passed or was skipped.
- **FAIL**: Any required reviewer reported FAIL.

If verification fails for a task:
- Present the failure details from each failing reviewer to the user.
- Reopen the task: `TaskUpdate(taskId: "<id>", status: "pending")`
- Update the task description by appending the failure details so the next worker has context:
  ```
  --- VERIFICATION FAILURE (attempt <N>) ---
  Correctness: <PASS/FAIL + details>
  Security: <PASS/FAIL + details>
  Quality: <PASS/FAIL + details>
  Visual: <PASS/FAIL/SKIP + details>
  ```
- The task will be picked up in the next wave with this additional context.
- Inform the user which tasks failed and why.
- If a task fails verification 3 times, escalate to the user and ask whether to skip it, modify the acceptance criteria, or debug manually.

#### 2.4 — Wave checkpoint and commit

When all tasks in a wave pass verification:
- Commit the wave's changes with a descriptive message: `feat(<component>): implement <summary> [tasks #X, #Y, #Z]`
- Tag the commit: `git tag wave-<slug>-<N>-complete` (where `<slug>` matches the plan file slug and N is the wave number within this implementation run).
- Push to the feature branch.
- Proceed to the next wave (back to Phase 1).

If a subsequent wave fails catastrophically (multiple tasks fail verification repeatedly), the user can roll back to the previous wave checkpoint:
```
git reset --hard wave-<slug>-<N-1>-complete
```
Present this option to the user if the situation arises — never execute it automatically.

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

## Prompt Templates

Worker and reviewer prompt templates are maintained in separate reference files for clarity:

- **Worker profiles:** `references/worker-profiles.md` — Four specialized worker prompts (backend, frontend, infrastructure, general) with profile selection logic based on task component metadata.
- **Reviewer profiles:** `references/reviewer-profiles.md` — Three parallel reviewer prompts (correctness, security, quality) plus the visual verification prompt.
- **Test writer prompt:** Defined inline in Phase 1.4 above — the test writer is always the same regardless of task type, since test writing is a single discipline.

When dispatching agents, read the appropriate reference file to get the current prompt template. Do not hardcode prompt text — always read from the reference file so that template updates take effect immediately.

## Conversation Style

- Be direct and operational. This is an execution phase, not a planning phase.
- Show progress clearly — use the wave/task structure to keep the user oriented.
- When environment setup is needed (Phase 0), be helpful and generate complete files — don't leave placeholders for the user to fill in.
- During dispatch (Phase 1), keep the user informed but don't ask for permission on every task — they already approved the plan.
- During manual verification (Phase 4), be thorough but efficient. Don't make the user repeat information — pull context from the PRD and design doc.
- When filing issues, generate complete, well-structured bug reports. The user should only need to confirm, not compose.
