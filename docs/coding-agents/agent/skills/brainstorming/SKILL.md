---
name: brainstorming
description: Use before planning or implementation when a feature, product behavior, architecture direction, workflow, documentation structure, or agent-harness design is unclear. Trigger when the user asks to brainstorm, explore options, clarify requirements, shape a feature, compare approaches, turn a vague idea into a spec, or decide what to build before writing an execution plan. Do not modify files or implement code during brainstorming unless the user explicitly switches to execution.
---

# Brainstorming

Use this skill to turn an unclear idea into a concrete, reviewable direction before planning or implementation.

Brainstorming protects intent. TDD Gate protects behavior during implementation. Review Gate checks the finished diff.

## Read Project Context

Before proposing directions, read only the context needed for the decision:

1. `AGENTS.md`
2. `README.md`
3. `docs/README.md` if present
4. `docs/project-status.md` if present
5. `docs/roadmap.md` if present
6. Relevant specs, ADRs, plans, docs, code, or tests named by the user

If the repository is unfamiliar and the question depends on existing code, do a read-only scout first.

## Ground Rules

- Do not edit files during brainstorming.
- Do not write implementation code.
- Do not jump straight to a single solution when the problem is under-specified.
- Ask at most three high-value questions at a time.
- Prefer concrete tradeoffs over abstract pros and cons.
- State assumptions explicitly.
- Keep non-goals visible.
- End with a decision-ready summary, not an execution diff.

## Brainstorming Loop

Use this loop until the direction is clear enough to plan:

```text
Understand goal
-> identify constraints and non-goals
-> ask focused questions
-> propose 2-3 viable approaches
-> compare tradeoffs and risks
-> recommend one direction with assumptions
-> draft acceptance criteria and open questions
-> get user confirmation before planning or implementation
```

Do not continue asking questions when a reasonable default is low-risk. State the assumption and proceed with a provisional recommendation.

## What To Explore

Cover the areas that matter for the decision:

- user goal and success criteria
- target users or readers
- non-goals and out-of-scope work
- existing project conventions
- affected files or modules
- blast radius and rollback path
- data, API, security, performance, deployment, or compatibility risks
- testing and validation strategy
- reviewable slices for later planning

## Option Quality Bar

Each serious option should include:

- core idea
- where it fits in the existing project
- likely files or docs affected
- benefits
- risks and failure modes
- validation approach
- whether it can be delivered in one slice or should be phased

Discard options that only differ cosmetically.

## Stop Conditions

Stop and ask for a decision when:

- product intent or user outcome is contradictory
- implementation would require choosing between materially different architectures
- the user needs to approve scope, dependency, migration, or public API changes
- risks cannot be evaluated without more project context
- the next step would be editing files or writing code

When stopped, present the smallest useful decision rather than a broad list of questions.

## Handoff

After brainstorming:

- If the user approves a direction, move to an Execution Plan or `new-feature-tdd`.
- If the task is still uncertain, run a read-only scout or solution comparison.
- If implementation starts, use TDD Gate for behavior changes.
- If the task is large or can be split, define reviewable slices before implementation.
- If using subagents later, give each subagent the confirmed task text, acceptance criteria, constraints, files, and validation commands.

## Output Format

```text
brainstorming-result:

Goal:
-

Assumptions:
-

Non-goals:
-

Options:
1. name:
   idea:
   benefits:
   risks:
   validation:

Recommendation:
-

Acceptance criteria:
-

Open questions:
-

Suggested next step:
- confirm direction / scout / write plan / start TDD slice:
```

## Safety Rules

- Do not present speculation as project fact.
- Do not use brainstorming approval as permission to implement.
- Do not hide tradeoffs behind a single confident recommendation.
- Do not expand scope after the user confirms a direction.
