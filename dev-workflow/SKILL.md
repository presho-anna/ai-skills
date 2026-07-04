---
name: dev-workflow
description: Run a 4-agent development workflow (Planner, Coder, Tester, Reviewer) with a shared .pipeline folder for inter-agent context.
---

# Dev Workflow

A multi-agent pipeline that takes a vague feature request through planning, implementation, testing, and review — each handled by a specialized agent on a different model, communicating via a shared `.pipeline/` folder.

## When to use

Invoke with `/dev-workflow` when you want a structured development process for a feature, bugfix, or refactor.

## Shared context: `.pipeline/`

Before starting, create a `.pipeline/` directory in the project root. Each agent writes its output there so the next agent can read it:

| File | Written by | Read by |
|------|-----------|---------|
| `.pipeline/spec.md` | Planner | Coder, Tester, Reviewer |
| `.pipeline/changes.md` | Coder | Tester, Reviewer |
| `.pipeline/tests.md` | Tester | Reviewer |
| `.pipeline/review.md` | Reviewer | User |

## Workflow

When the user provides a task description, execute these steps in order:

### Step 0: Setup

- Create `.pipeline/` in the project root
- Ensure `.pipeline/` is listed in `.gitignore`
- Record the current HEAD sha (store in `.pipeline/baseline.sha`) — the Reviewer diffs against this, not the working tree

### Step 1: Planner (Opus)

Spawn an agent with `model="opus"`:
- Read the existing project structure, relevant source files, and any CLAUDE.md / README first
- Take the vague feature request and turn it into a detailed spec
- Match existing naming conventions, patterns, and directory structure
- The spec MUST include:
  - Exact file paths to create or modify
  - Function signatures (name, params, return type)
  - Edge cases and how to handle each one
  - Acceptance criteria
  - Test file paths and the test framework/runner to use
- Write the spec to `.pipeline/spec.md`

**Gate:** After writing the spec, pause and show the user a summary. Only proceed to Step 2 when the user approves.

### Step 2: Coder (Sonnet)

Spawn an agent with `model="sonnet"`:
- Read `.pipeline/spec.md`
- Build exactly what the spec says — no improvisation, no extras
- Write production-ready code (no placeholders or TODOs)
- Make actual file edits
- Write a summary of all changes (files modified, functions added/changed) to `.pipeline/changes.md`

### Step 3: Tester (Sonnet)

Spawn an agent with `model="sonnet"`:
- Read `.pipeline/spec.md`, `.pipeline/changes.md`, and the actual code the Coder wrote
- Write test cases covering:
  - Happy path for each function/feature
  - Every edge case identified in `.pipeline/spec.md`
- Place tests at the paths specified in the spec
- Make actual test file edits
- Write a summary of test coverage to `.pipeline/tests.md`

### Step 4: Reviewer (Opus)

Spawn an agent with `model="opus"`:
- **READ-ONLY — this agent MUST NOT edit any code or test files**
- Read `.pipeline/spec.md`, `.pipeline/changes.md`, `.pipeline/tests.md`
- Read the actual source and test files
- Run a diff against the baseline sha (`git diff <baseline>`) to see exactly what changed
- Give a verdict:
  - Does the code match the spec?
  - Are there correctness, performance, or security issues?
  - Are the tests sufficient?
  - Pass / Fail with specific reasons
- Write the verdict to `.pipeline/review.md`
- If issues are found, summarize them for the user and ask whether to iterate

## Iteration

When the Reviewer gives a **Fail** verdict:
1. Feed `.pipeline/review.md` back to the Coder as additional context
2. The Coder re-reads the spec + review feedback, makes fixes, updates `.pipeline/changes.md`
3. Re-run Tester (updates `.pipeline/tests.md` if needed)
4. Re-run Reviewer (overwrites `.pipeline/review.md` with new verdict)

Repeat until Pass or the user decides to stop.

## Rules

- Each agent writes to `.pipeline/` — this is how context flows between agents
- Agents run sequentially; each depends on the previous agent's output
- The Planner must explore the codebase before writing the spec
- The user approves the spec before implementation begins
- The Reviewer is strictly read-only — it gives a verdict, never fixes code
- `.pipeline/` persists after completion for reference context
- Report a brief summary after each step completes so the user can follow progress
- Keep each agent's prompt focused on its role and the files it needs to read
