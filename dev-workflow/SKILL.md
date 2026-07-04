---
name: dev-workflow
description: Run a sized, gated multi-agent development workflow (Planner, Coder, Tester, Reviewer) with a shared .pipeline/ folder and an append-only decision log. Use this whenever the user invokes /dev-workflow, hands over a Jira ticket, bug report, feature request, or refactor task, or asks for a "structured" or "pipelined" implementation. Also use it when the user asks to resume, iterate on, or review a previous .pipeline/ run.
---

# Dev Workflow

A multi-agent pipeline that takes a ticket through planning, implementation, testing, and review — sized to the ticket, gated by the user, and leaving an audit trail in a shared decision log.

Two principles govern everything below:

1. **Effort is budgeted, not open-ended.** Every agent gets explicit read/time budgets and stop conditions. Exhausting a budget means *stop and ask*, never *keep reading*.
2. **Nothing diverges silently.** Any decision that isn't literally in the ticket — a rename, a pattern choice, a deviation from spec, a discovered hazard — goes in `.pipeline/decisions.md`. An undocumented divergence is a review failure.

## Shared context: `.pipeline/`

| File                     | Written by            | Read by                 |
| ------------------------ | --------------------- | ----------------------- |
| `.pipeline/spec.md`      | Planner               | Coder, Tester, Reviewer |
| `.pipeline/decisions.md` | **All agents** (append-only) | All agents, User  |
| `.pipeline/changes.md`   | Coder                 | Tester, Reviewer        |
| `.pipeline/tests.md`     | Tester                | Reviewer                |
| `.pipeline/review.md`    | Reviewer              | User                    |
| `.pipeline/baseline.sha` | Setup                 | Reviewer                |

Decision log format and entry types: read `references/decision-log.md` before Step 0, and include its rules in every agent's prompt.

## Step 0: Setup

- Create `.pipeline/` in the project root; ensure it is in `.gitignore`
- Record current HEAD: `git rev-parse HEAD > .pipeline/baseline.sha`
- Create `.pipeline/decisions.md` from the template in `references/decision-log.md`

## Step 0.5: Sizing gate (before spawning any agent)

Read the ticket and classify it. State the size and why in your progress summary. When in doubt between two sizes, pick the smaller — the spec-approval gate catches undersizing cheaply; oversizing burns 17 minutes on a rename.

| Size | Signals | Pipeline shape | Planner read budget |
| ---- | ------- | -------------- | ------------------- |
| **S** | Single-file or few-line change; clear repro or exact location named in ticket; no design decisions | No Planner agent. Orchestrator writes a mini-spec (≤ 15 lines: files, change, edge cases, test location) directly into `spec.md`, gates with user, then one Sonnet agent does Coder+Tester combined. Reviewer still runs. | ≤ 5 files |
| **M** | Touches 2–5 files; one subsystem; some design choice but within existing patterns | Full 4-agent pipeline, standard budgets | ≤ 15 files |
| **L** | New subsystem, cross-cutting change, migration, or ambiguous scope | Full pipeline, extended budgets, **mandatory** clarifying-question round before the spec | ≤ 30 files |

## Step 1: Planner (Opus)

Spawn an agent with `model="opus"`. Its prompt is the full protocol in `references/planner.md` — read that file and include it in the subagent prompt, **replacing the `{READ_BUDGET}` placeholder with the number from the sizing table above** (this is a manual fill-in, not automatic substitution), then append the ticket and the size class.

The short version of what that protocol enforces:

- **Triage unknowns before exploring.** Repo-answerable questions get targeted searches; user-only questions (intent, scope, priorities) get asked at the gate — max 5. Never explore to compensate for an ambiguous ticket.
- **Search-first, budgeted discovery.** Grep/glob from ticket keywords; read only hits; count every file read against the budget; one exemplar file per convention, not the whole namespace.
- **Fenced-off paths.** No dependency source (gems, `node_modules/`, `vendor/`), no lockfiles, no generated files, no second examples of an already-understood pattern.
- **Stop condition.** The moment the Planner can name the files to change, the pattern to follow, and where tests go — it stops and writes the spec. Budget exhausted with questions remaining → stop and ask, don't keep reading.

The spec MUST still include: exact file paths to create/modify, function signatures, edge cases and handling, acceptance criteria, test file paths and runner.

**Gate:** Show the user (a) a spec summary, (b) the Planner's open questions, (c) new `decisions.md` entries, and (d) the Planner's budget report (files read / budget, time). Proceed only on approval. If the user answers questions, the Planner amends the spec and logs each answer-driven choice as a decision — it does not re-explore.

## Step 2: Coder (Sonnet)

Spawn an agent with `model="sonnet"`:

- Read `.pipeline/spec.md` and `.pipeline/decisions.md`
- Build exactly what the spec says — production-ready, no placeholders
- **If reality forces a deviation from the spec** (spec path doesn't exist, signature can't work, a name collides): log a `DIVERGENCE` entry in `decisions.md` *before* implementing the workaround, and flag it in `changes.md`. Silent improvisation is prohibited.
- Discover something dangerous or surprising (fragile coupling, dead code that isn't, a footgun)? Log it as `HEREBEDRAGONS`.
- Write a summary of all changes to `.pipeline/changes.md`

## Step 3: Tester (Sonnet)

Spawn an agent with `model="sonnet"`:

- Read `spec.md`, `changes.md`, `decisions.md`, and the actual code
- Cover the happy path and every edge case in the spec; place tests at the spec's paths; run them
- A test that can only pass by contradicting the spec → log `DIVERGENCE`, do not quietly adjust the assertion
- Write coverage summary to `.pipeline/tests.md`

## Step 4: Reviewer (Opus)

Spawn an agent with `model="opus"` — **strictly read-only** (verdicts, never fixes):

- Read all `.pipeline/` files, the source, the tests; diff with `git diff $(cat .pipeline/baseline.sha)`
- Verdict on: spec conformance; correctness/performance/security; test sufficiency
- **Decision-log audit:** walk the diff against the spec. Every behavioral or naming difference must have a `decisions.md` entry. Any undocumented divergence = **Fail** with the entry that should have existed.
- Write verdict to `.pipeline/review.md`; if Fail, summarize for the user and ask whether to iterate

## Iteration

On **Fail**: feed `review.md` to the Coder → Coder fixes and updates `changes.md` (logging any new divergences) → re-run Tester → re-run Reviewer. Repeat until Pass or the user stops. `decisions.md` is never rewritten between iterations — only appended.

## Rules

- Agents run sequentially; context flows only through `.pipeline/`
- The user approves the spec before implementation begins
- `decisions.md` is append-only; every agent reads it on start and may append; only the format in `references/decision-log.md` is valid
- Budgets are hard limits: an agent that hits one reports state and stops; the orchestrator asks the user before granting more
- Report a brief summary (including any new decision entries) after each step
- `.pipeline/` persists after completion — it is the record of what was decided and why
