---
name: writing-plans
description: Use after requirements, brainstorming, or a Spec are clear and before implementation for a multi-step feature, bug fix, refactor, migration, documentation restructure, or agent-harness change. Trigger when the user asks to write an implementation plan, execution plan, plan document, reviewable slices, task breakdown, or docs/plans entry. Produce a concrete docs/plans/*.md plan with scope, non-goals, blast radius, affected files, TDD steps, validation commands, rollback, and plan self-review. Do not implement code while writing the plan.
---

# Writing Plans

Use this skill to turn a confirmed direction or Spec into an implementation plan that a low-context implementer agent can execute without redesigning the task.

Brainstorming protects intent. Writing Plans controls action. TDD Gate protects behavior during implementation. Review Gate checks the finished diff.

## Read Project Rules

Before writing a plan, read these if present:

1. `AGENTS.md`
2. `README.md`
3. `docs/README.md`
4. `docs/project-status.md`
5. `docs/roadmap.md`
6. `docs/development.md`
7. `docs/testing.md`
8. `docs/agent/playbooks/principles/task-document-layers.md`
9. `docs/agent/playbooks/principles/reviewable-slices.md`
10. Relevant Spec, ADR, issue, bug report, prior brainstorming result, code, and tests

Project-local rules override this skill when they are stricter.

## Preconditions

Write an Execution Plan only when the goal, non-goals, and acceptance criteria are clear enough to constrain implementation.

If requirements are vague, stop and use Brainstorming first.

If the repository area is unfamiliar, do a read-only scout before planning.

If the task spans independent subsystems, propose separate plans. Each plan should produce a testable, reviewable result on its own.

## Plan Destination

Default destination:

```text
docs/plans/YYYY-MM-DD-feature-name.md
```

If the project has another documented plans directory, use that instead.

Do not create a plan file until the user wants a written plan. If the user only asks for a draft in chat, provide the same structure without editing files.

## Planning Workflow

Use this sequence:

```text
Confirm goal and non-goals
-> map relevant project context
-> map affected files and responsibilities
-> assess blast radius
-> split into reviewable slices
-> define TDD and validation for each slice
-> define rollback and stop conditions
-> self-review for coverage, placeholders, and consistency
-> ask user to confirm before implementation
```

Do not implement while planning.

## File And Boundary Map

Before writing slices, list expected file changes:

- exact files to create
- exact files to modify
- exact tests to create or modify
- docs/status files to update
- files or directories that are explicitly out of scope

For each file, state its responsibility. Follow existing project patterns. Do not plan broad directory reshuffles unless the task is explicitly a restructure.

## Reviewable Slice Rules

Each slice must:

- represent one behavior, migration step, documentation unit, or refactor boundary
- be small enough for a 5-10 minute human review
- have a focused validation command or documented alternative proof
- include TDD steps when behavior changes
- name expected files
- include rollback instructions
- stop after completion for review when risk is meaningful

Prefer fewer, stronger slices over many cosmetic microtasks. Use micro-steps inside each slice only where they prevent TDD or validation shortcuts.

## Required Plan Sections

Use the base plan shape from `docs/agent/playbooks/templates/project-harness-files.md` when present. This skill does not replace that template; it strengthens it by requiring file responsibilities, stop conditions, and plan self-review.

Every plan should include the base sections plus the strengthened fields below:

````markdown
# Feature Name Implementation Plan

Date: YYYY-MM-DD

## Goal

## Non-goals

## Linked Context

- Spec:
- Roadmap:
- ADR / decision:
- Issue / bug report:

## Spec Summary

## Blast Radius

| Area | Impact? | Notes |
|---|---|---|
| Public API |  |  |
| Data model |  |  |
| Auth / security |  |  |
| Network / external services |  |  |
| Deployment / config |  |  |
| Test fixtures |  |  |

## Affected Files

| File | Create / Modify / Test / Docs | Responsibility | Expected change |
|---|---|---|---|

## Acceptance Criteria

- [ ]

## Reviewable Slices

### Slice 1: Name

Status: todo / doing / done / blocked

Goal:

Files:

Tests:

Steps:

- [ ] Write failing test:
- [ ] Run focused test and confirm RED for expected reason:
- [ ] Write minimal implementation:
- [ ] Run focused test and confirm GREEN:
- [ ] Refactor only if needed:
- [ ] Run validation:
- [ ] Update plan/status docs:

Validation commands:

```bash

```

Rollback:

Stop conditions:

## Full Validation

## Risks

## Done Criteria

## Plan Change Log
````

For docs-only or config-only tasks, replace TDD steps with the proportionate proof, such as link checks, render checks, focused config command, or review checklist.

## No Placeholders

These are plan failures:

- `TODO`, `TBD`, `later`, `fill in details`
- "handle edge cases" without naming the edge cases
- "add proper validation" without saying what must be validated
- "write tests" without naming the behavior and test location
- "similar to previous slice" instead of repeating the needed details
- broad tasks that mix feature, refactor, formatting, dependency, and docs changes
- validation commands without expected outcome
- references to functions, files, types, APIs, or docs not introduced or linked in the plan

If a detail is unknown, write the exact unknown and the scout or decision needed before implementation.

## Plan Self-Review

Before handing off, run this review yourself:

1. **Spec coverage:** every acceptance criterion maps to at least one slice.
2. **Scope control:** non-goals are not planned as work.
3. **File consistency:** affected files match slice steps.
4. **TDD coverage:** every behavior change has a failing-test step or documented alternative proof.
5. **Validation:** every slice has a focused command and the plan has full validation.
6. **Reviewability:** each slice is small enough to review.
7. **Rollback:** risky slices say how to revert or disable the change.
8. **Placeholder scan:** no vague placeholders remain.
9. **Name consistency:** file names, functions, types, commands, and doc paths are consistent across slices.

Fix plan issues before asking the user to approve implementation.

## Handoff

After writing the plan, ask for confirmation before execution.

Recommended handoff:

```text
Plan written: docs/plans/YYYY-MM-DD-feature-name.md

Recommended execution:
1. Execute Slice 1 only with tdd-gate.
2. Run focused validation and ./scripts/check when appropriate.
3. Run review-gate.
4. Update plan status and docs/project-status.md.

Please confirm before implementation begins.
```

If subagents are available and the plan has independent slices, mention that subagent-driven execution is possible. Do not require it by default.

## Safety Rules

- Do not implement code while writing the plan.
- Do not use a plan as permission to change files beyond the plan.
- Do not plan commits as mandatory execution steps unless the user asked for commit workflow.
- Do not bury open product or architecture decisions inside implementation steps.
- Do not overwrite existing plans; update or append a change log unless the user asks for replacement.
- Do not include secrets, credentials, machine-specific paths, or local-only runtime details in plan files.
