# Implementation Report — {Product Name}

**Date:** {date}
**Branch:** implement/{slug}
**Plan:** [plan-{slug}.md](../docs/plan-{slug}.md)
**Design Doc:** [design-{slug}.md](../docs/design-{slug}.md)
**PRD:** [prd-{slug}.md](../docs/prd-{slug}.md)
**CI Status:** {✅ Passing / ❌ Failing}
**Commit Range:** {base_sha}..{head_sha}
**Wave Checkpoints:** {wave-<slug>-1-complete, wave-<slug>-2-complete, ...}

---

## Summary

| Metric | Value |
|--------|-------|
| Tasks completed | {completed}/{total} |
| Waves executed | {wave_count} |
| Files changed | {files_changed} |
| Lines added | {lines_added} |
| Lines removed | {lines_removed} |
| Tests passing | {test_count} |
| Tests written (TDD) | {tdd_test_count} |
| Test coverage | {coverage}% |
| Retry count | {tasks_retried} tasks needed re-dispatch |
| Security issues found | {security_issues} (all resolved before merge) |
| Visual checks | {visual_pass}/{visual_total} passed ({visual_skip} skipped) |

---

## Execution Log

### Wave 1: {description}
| Task ID | Subject | Worker Profile | Status | Attempts | Correctness | Security | Quality | Visual | Notes |
|---------|---------|---------------|--------|----------|-------------|----------|---------|--------|-------|
| {id} | {subject} | backend | ✅ | 1 | ✅ | ✅ | ✅ | N/A | — |
| {id} | {subject} | frontend | ✅ | 2 | ✅ | ❌→✅ | ✅ | ✅ | Security: XSS in input handler. Fixed on retry. |

### Wave 2: {description}
| Task ID | Subject | Worker Profile | Status | Attempts | Correctness | Security | Quality | Visual | Notes |
|---------|---------|---------------|--------|----------|-------------|----------|---------|--------|-------|
| {id} | {subject} | general | ✅ | 1 | ✅ | ✅ | ✅ | N/A | — |

{Repeat for each wave.}

---

## Manual Verification Results

| # | Check | Source | Result | Issue |
|---|-------|--------|--------|-------|
| 1 | {description} | PRD §{section} | ✅ Pass | — |
| 2 | {description} | Design §{section} | ❌ Fail | #{issue_number} |
| 3 | {description} | Design §{section} | ⏭ Skip | #{issue_number} |

---

## Issues Filed

| Issue | Title | Type | Severity | Related Tasks |
|-------|-------|------|----------|---------------|
| #{number} | {title} | bug | {severity} | #{task_ids} |
| #{number} | {title} | needs-testing | — | #{task_ids} |

---

## Environment Setup

Files created or modified during Phase 0:
- {`scripts/install_pkgs.sh` — created / already existed}
- {`.claude/settings.json` — SessionStart hooks added / already configured}
- {`.github/workflows/ci.yml` — created / already existed}

---

## Next Steps

- [ ] Open PR: `implement/{slug}` → `main`
- [ ] Code review
- [ ] Address filed bug issues (#{numbers})
- [ ] Complete skipped verification items (#{numbers})
- [ ] {any other follow-ups from the implementation}
