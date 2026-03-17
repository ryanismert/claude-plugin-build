# Design Spec: Phase 4 Browser-Driven Verification

**Date:** 2026-03-17
**Skill:** `/build:implement`
**Scope:** Replace Phase 4 (Manual Verification & Issue Filing) with agent-driven browser verification

## Problem

Phase 4 currently requires the user to manually walk through each verification checklist item, report pass/fail/skip, and then the orchestrator files GitHub issues for failures. This is slow, tedious, and inconsistent with the rest of the skill which uses agent-driven automation.

## Design

### Structural Change

Phase 4 collapses from 5 sub-phases to 3:

| Sub-phase | Current | Redesigned |
|-----------|---------|------------|
| 4.1 | Generate checklist | Generate classified checklist (browser-verifiable vs manual-only) |
| 4.2 | Walk user through items interactively | Spawn browser agent to execute browser-verifiable items |
| 4.3 | File GitHub issues per failure | Removed |
| 4.4 | Handle skipped items, file tracking issues | Removed |
| 4.5 | Save completion report | Final report with all results + opt-in issue filing offer |

### Phase 4.1: Classified Checklist Generation

Same sources as before (PRD acceptance criteria, design doc UX flows, E2E scenarios, cross-browser, performance, security). Each item is now:

- **Tagged by type:** `flow`, `interaction`, `visual`, `performance`, `security`, `external`
- **Classified:** browser-verifiable or manual-only
- **Ordered:** browser-verifiable items arranged into dependency chains (e.g., registration before auth-dependent flows)

Only items relevant to actually-implemented tasks are included (unchanged from current behavior).

### Phase 4.2: Browser Agent Execution

A single browser verification agent is spawned with the full ordered list of browser-verifiable items. The agent executes items sequentially in a single browser session, building on prior state (accounts created, sessions established).

**Browser tools:** The agent requires Chrome DevTools MCP or equivalent browser automation tools. Detection uses the same check as Phase 2.2 (Chrome DevTools MCP, Playwright).

**Per-item behavior:**

1. Navigate to the relevant page/starting point
2. Execute multi-step interactions (fill forms, click buttons, follow redirects)
3. At key checkpoints: take snapshots (accessibility tree) and/or screenshots
4. Check browser console for errors
5. Evaluate outcome against acceptance criteria
6. If **PASS** — move to next item
7. If **FAIL** — evaluate using two-part fix test:
   - Do the acceptance criteria unambiguously specify the correct behavior?
   - Is the fix a localized code change (fewer than ~20 lines, single file)?
   - If BOTH true: attempt fix, wait for hot-reload, re-verify, run tests for regressions, commit as `fix: <description> [phase-4-verification]`
   - If either false: report as FAIL with details, do not attempt fix
8. If **ERROR** (page didn't load, browser tool failure): report as ERROR, skip downstream dependent items

**Result statuses:**
- `PASS` — worked as expected
- `PASS (fixed)` — failed initially, agent fixed it, now passes; includes what was changed and commit reference
- `FAIL` — couldn't be fixed or fix didn't work; includes details and screenshots
- `ERROR` — agent couldn't complete the verification
- `SKIPPED` — blocked by prior failure or no browser tools
- `MANUAL` — item requires manual verification (non-browser-verifiable)

**Graceful degradation:**
- If no browser tools are available (no Chrome DevTools MCP, no Playwright), all browser-verifiable items are reported as `SKIPPED — no browser tools configured` with a clear message to the user
- If an item fails in a way that breaks subsequent dependent items, downstream items are marked `SKIPPED — blocked by item #N failure`

**Early abort:** If the first 3 items all result in ERROR (not FAIL), abort browser verification entirely and report the root cause to the user.

**Dev server lifecycle:** The agent starts the dev server if not already running. After making code fixes, it waits for hot-reload or restarts the server if hot-reload is not available.

### Phase 4.3: Final Report

**Post-fix test run:** If any fixes were applied during Phase 4.2, the full test suite is run before generating the report. CI Status and test counts reflect the post-fix state.

The completion report (`docs/implement-<slug>.md`) includes:

**Verification results table** with all items (browser-verified, manual-only, skipped):

| # | Check | Type | Result | Details |
|---|-------|------|--------|---------|
| 1 | User registration flow | flow | PASS | Signup, redirect, session verified |
| 2 | Auth token refresh | flow | PASS (fixed) | Fixed redirect URL in auth callback |
| 3 | Mobile responsiveness | visual | FAIL | Dashboard sidebar overlaps content at 375px |
| 4 | Email delivery | external | MANUAL | Requires manual verification |
| 5 | Error states | interaction | SKIPPED | Blocked by item #1 failure |

**Fixes applied** section listing each fix with file, line, what was changed, and commit hash.

**Opt-in issue filing:** After presenting the report, the orchestrator asks: "Want me to file GitHub issues for any of the failures?" If the user says yes, it drafts issues for the FAIL items and shows them for approval before filing. If the user declines, the report stands on its own.

## Files Changed

- `skills/implement/SKILL.md` — Replace Phase 4 sections (4.1–4.5) with new 4.1–4.3; update skill description in frontmatter; update Prompt Templates section; update Conversation Style section

## Not Changed

- Phase 2.2 visual verification — remains as-is (component-level, report-only, per-task)
- Phase 2 reviewer profiles — remain report-only (no fix attempts)
- `references/reviewer-profiles.md` — no changes
- `references/worker-profiles.md` — no changes
