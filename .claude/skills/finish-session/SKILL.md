---
name: finish-session
description: Update task tracker, update plan status, write progress log, commit changes.
disable-model-invocation: true
---

# Finish Session

Wrap up the session — track progress and commit.

## Step 1: Update task tracker
- Read current tasks via `TaskList`
- Mark completed tasks as `completed` via `TaskUpdate`
- Verify no task left as `in_progress` without completion

## Step 2: Update plan status
- Read plan overview: `plans/260318-hypertrader-v2-etap1/plan.md`
- Update phase status from `pending` to `done` for completed phases

## Step 3: Update progress log
- Append a new entry to `plans/260318-hypertrader-v2-etap1/progress.md`:
  - **Date**: current date
  - **Tasks completed**: list of finished tasks
  - **Files created/modified**: list
  - **Decisions made**: architecture choices, trade-offs
  - **Issues**: problems encountered, how resolved
  - **Next session**: what to work on next

## Step 4: Commit
- !`git status --short`
- Stage all relevant changes (NOT .env, secrets, __pycache__, .venv)
- Create commits following conventions:
  - Imperative mood, focus on "why" not "what"
  - If multiple logical changes — split into separate commits
  - Format: `feat: ...`, `fix: ...`, `refactor: ...`, `docs: ...`

## Step 5: Phase transition check
Check tasks: if ALL tasks in the current phase are completed:
- Notify the user that the phase is done
- Show next unblocked tasks from task list
- Suggest what to tackle next session

## Step 6: Summary
Report to user:
- Tasks marked as done
- Commits created
- Next pending task from tracker
- Overall progress (X/9 phases done)
- Phase transition status (if applicable)
