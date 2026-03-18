---
name: start-session
description: Initialize a working session — read context, pick next task, consult expert agents, build implementation plan.
disable-model-invocation: true
---

# Start Session

Execute the session start protocol with expert agent consultation.

## Step 1: Project context
- Read `PLAN.md` (project spec and architecture)
- Read `CLAUDE.md` (rules, structure, agents, workflows)

## Step 2: Find current work
- Current branch: !`git branch --show-current`
- Recent commits: !`git log --oneline -10 2>/dev/null || echo "no commits yet"`
- Working tree: !`git status --short 2>/dev/null || echo "no git"`
- Read task list via `TaskList` tool
- Read plan overview: glob `plans/*/plan.md` (latest plan)
- Read progress log: glob `plans/*/progress.md` (if exists)

## Step 3: Pick next task
Scan tasks in order to find the first unblocked pending task:
1. If pending unblocked task found — read its phase file from plan directory
2. If ALL tasks done — go to Step 3B

### Step 3B: Start new phase
When no pending tasks exist:
1. Review `PLAN.md` for next stages
2. Present options to user via `AskUserQuestion` — let them choose
3. Create new plan with phases and tasks
4. The first task becomes the session target

## Step 4: Understand the task
Before launching agents, YOU (Claude main) must:
- Read all existing code relevant to the task (models, schemas, related modules)
- Read the phase plan file for implementation details
- Read reference code from `../BITCOIN/src/` if phase file references it
- Understand the task requirements, dependencies, and constraints
- Determine which expert agents are relevant (see Agent Selection below)

## Step 5: Consult expert agents
Launch **relevant** agents via the Agent tool with `run_in_background: true`.
Launch ALL selected agents in a **single message** so they run concurrently.

Each agent receives:
- The task description from the phase plan
- Relevant context (existing code, architecture decisions, project conventions)
- A specific question: "How would you approach implementing this task?"

### Agent Selection Criteria
Select agents based on task nature. **Not all agents are needed for every task.**

| Agent | When to include | When to skip |
|-------|----------------|--------------|
| `planner` | Complex task needing decomposition, architecture decisions | Task already fully decomposed in phase plan |
| `researcher` | New API, unfamiliar tech, need latest docs | Well-understood task, research already done |
| `feature-dev:code-architect` | Module structure, patterns, interfaces, data flow | Trivial config/setup changes |
| `debugger` | Investigating existing bug or runtime issue | New feature implementation |

**Minimum**: at least 2 agents for any non-trivial task.
**Skip all agents**: only for trivial setup/config tasks (e.g., project scaffolding).

## Step 6: Clarify before planning
- Collect all agent recommendations
- Identify **consensus** (where agents agree) and **divergence** (where they disagree)
- Identify **ambiguities** or **decisions** that require user input
- Use `AskUserQuestion` for ALL unresolved questions — do NOT proceed with assumptions
- Resolve every question BEFORE entering plan mode

## Step 7: Plan
- Call `EnterPlanMode` to switch into plan mode
- Write a consolidated implementation plan:
  - Approach summary
  - Files to create/modify
  - Key design decisions (with agent consensus/divergence noted)
- Call `ExitPlanMode` — user reviews and approves the plan

## Step 8: Implement
- Mark task as `in_progress` via `TaskUpdate`
- Follow the approved plan step by step
- After each file creation/modification — verify syntax: `python3 -c "import ast; ast.parse(open('<file>').read())"`
- Keep files under 200 lines
- Follow EventBus patterns as defined in architecture
- Follow YAGNI/KISS/DRY principles
