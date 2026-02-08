# Implementation Report — {Product Name}

**Date:** {date}
**Branch:** implement/{slug}
**Plan:** [plan-{slug}.md](../docs/plan-{slug}.md)
**Design Doc:** [design-{slug}.md](../docs/design-{slug}.md)
**PRD:** [prd-{slug}.md](../docs/prd-{slug}.md)
**CI Status:** {✅ Passing / ❌ Failing}
**Commit Range:** {base_sha}..{head_sha}

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
| Test coverage | {coverage}% |
| Retry count | {tasks_retried} tasks needed re-dispatch |

---

## Execution Log

### Wave 1: {description}
| Task ID | Subject | Status | Attempts | Notes |
|---------|---------|--------|----------|-------|
| {id} | {subject} | ✅ | 1 | — |
| {id} | {subject} | ✅ | 2 | Failed verification: {reason}. Fixed on retry. |

### Wave 2: {description}
| Task ID | Subject | Status | Attempts | Notes |
|---------|---------|--------|----------|-------|
| {id} | {subject} | ✅ | 1 | — |

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
