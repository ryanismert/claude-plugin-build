# {Product Name} — Implementation Plan

**Date:** {date}
**Status:** Active
**Source Design Doc:** [{design doc filename}]({relative path to design doc})
**Source PRD:** [{prd filename}]({relative path to prd})
**Task List ID:** `{CLAUDE_CODE_TASK_LIST_ID value}`

---

## Task Graph

### Phase 1: {Phase Name}
{Description of what this phase accomplishes.}

| Task ID | Subject | Complexity | Blocked By | Parallelizable |
|---------|---------|------------|------------|----------------|
| {id} | {subject} | {S/M/L} | {— or list of IDs} | {Yes / No} |

### Phase 2: {Phase Name}
{Description of what this phase accomplishes.}

| Task ID | Subject | Complexity | Blocked By | Parallelizable |
|---------|---------|------------|------------|----------------|
| {id} | {subject} | {S/M/L} | {list of IDs} | {Yes / No} |

### Phase 3: {Phase Name}
{Description of what this phase accomplishes.}

| Task ID | Subject | Complexity | Blocked By | Parallelizable |
|---------|---------|------------|------------|----------------|
| {id} | {subject} | {S/M/L} | {list of IDs} | {Yes / No} |

{Add additional phases as needed.}

---

## Critical Path

The longest sequential chain of dependencies:

```
{Task ID: Subject} → {Task ID: Subject} → {Task ID: Subject} → ...
```

Estimated critical path duration: {sum of complexity estimates}

---

## Summary

| Metric | Value |
|--------|-------|
| Total tasks | {count} |
| Phase 1 tasks | {count} |
| Phase 2 tasks | {count} |
| Phase 3 tasks | {count} |
| Small (S) | {count} |
| Medium (M) | {count} |
| Large (L) | {count} |
| Max parallelism | {max tasks runnable concurrently} |

---

## Execution

### Starting Work

Tasks are loaded in Claude Code's native task system. To begin:

1. **View tasks:** Press `Ctrl+T` in Claude Code or ask "show me all tasks"
2. **Execute tasks:** Run `/build:implement` to start executing unblocked tasks
3. **Multi-session:** Additional sessions can join with:
   ```
   CLAUDE_CODE_TASK_LIST_ID={task_list_id} claude
   ```
4. **Remote execution:** Hand off to remote workers by setting the same `CLAUDE_CODE_TASK_LIST_ID` in the remote environment

### Parallel Execution

{Describe which phases/tasks can be run in parallel and recommended session distribution.}

### Open Risks

| Risk | Impact | Mitigation | Affects Tasks |
|------|--------|------------|---------------|
| {risk from design doc} | {H/M/L} | {mitigation} | {task IDs} |
