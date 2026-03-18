---
name: verify-session
description: Review implementation with expert agents, apply fixes, simplify code, run all checks.
disable-model-invocation: true
---

# Verify Session

Review and polish the implementation before finishing.

## Context
- Changed files: !`git diff --name-only`
- Staged files: !`git diff --cached --name-only`
- Current branch: !`git branch --show-current`

## Step 1: Expert review
Launch **relevant** review agents via the Agent tool with `run_in_background: true`.
Launch ALL selected agents in a **single message** so they run concurrently.

### Agent Selection
Select agents based on what was implemented:

| Agent | When to include | When to skip |
|-------|----------------|--------------|
| `planner` | Architecture changes, new module patterns, data flow | Simple bug fixes |
| `feature-dev:code-reviewer` | **Always** — bugs, security, logic, conventions | Never skip |
| `debugger` | Performance concerns, async issues, race conditions | No runtime behavior changes |

**Minimum**: `code-reviewer` always runs. Add domain agents based on what changed.

Each agent receives:
- List of changed files with their full content
- Task context from the phase plan file
- Project conventions (Python 3.12, async, Pydantic v2, YAGNI/KISS/DRY)

## Step 2: Report findings
Present all findings organized by priority:
1. **Critical** — must fix (bugs, security, data integrity)
2. **Important** — should fix (logic, quality, conventions)
3. **Suggestion** — optional (style, naming)

Include specific file paths and line numbers.

## Step 3: Apply fixes
- Apply **critical** and **important** findings
- Present changes to user for approval
- Do NOT apply suggestions unless user requests it

## Step 4: Code simplification
After fixes are approved, launch `code-simplifier` agent:
- Focus on recently changed files only
- Simplify without changing behavior
- Apply only clear improvements
- Present simplifications to user

## Step 5: Verification checks
Run all applicable checks:
- Python syntax: `python3 -c "import ast; ast.parse(open('<file>').read())"` for each changed .py file
- Python imports: `source .venv/bin/activate && python -c "import src.<module>"` for each new/changed module
- Type check: verify Pydantic models instantiate correctly
- If anything fails — fix and re-run

## Step 6: Final report
- Verification results (pass/fail for each check)
- Review findings applied (list)
- Review findings deferred (and why)
- Code simplifications made (list)
- Ready for `/finish-session`
