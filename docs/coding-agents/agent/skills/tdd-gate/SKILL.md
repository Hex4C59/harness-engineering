---
name: tdd-gate
description: Use before implementing a feature, bug fix, behavior change, risky refactor, or execution-plan slice that should be test-driven. Trigger when the user says TDD, test first, regression test, failing test, red-green-refactor, executes a plan slice, fixes a bug, changes business logic, changes public APIs, or asks whether implementation may begin. Enforce RED-GREEN-REFACTOR with fresh validation evidence, while following project AGENTS.md and docs/testing.md first.
---

# TDD Gate

Use this skill to stop implementation from starting or continuing without a real failing test and a proportionate validation path.

TDD Gate protects behavior during implementation. Review Gate checks the finished diff. Commit Gate checks the project-history boundary.

## Read Project Rules

Before acting, read these if present:

1. `AGENTS.md`
2. `docs/testing.md`
3. `docs/development.md`
4. `docs/project-status.md`
5. `docs/agent/playbooks/principles/tdd-and-verification.md`
6. `docs/agent/playbooks/workflows/everyday-development.md`
7. The current Spec, Execution Plan, issue, bug report, or user request

Project-local rules override this skill when they are stricter.

## Decide Applicability

Use TDD by default for:

- new behavior or feature slices
- bug fixes and regressions
- pure logic
- business rules and workflow changes
- public API, parser, validation, persistence, auth, permissions, billing, networking, concurrency, or data-model changes
- refactors where behavior could accidentally change

Use a documented alternative when TDD is not the right primary proof:

| Change | Minimum proof |
|---|---|
| Docs-only | Link/render check or focused review |
| Small config/tooling change | Focused command showing the config is loaded |
| Visual-only UI | Component test, screenshot/browser verification, or explicit manual verification record |
| Prototype spike | Mark as spike; do not treat as final implementation until tests are added |
| Existing untested legacy code | Add a characterization or regression test at the safest seam before changing behavior |

If uncertain, do a lightweight TDD pass.

## Required Cycle

Run one small cycle per behavior:

```text
Identify behavior
-> RED: write one minimal failing test
-> Verify RED: run focused test and confirm the expected failure reason
-> GREEN: write the smallest implementation that can pass
-> Verify GREEN: run the focused test and confirm it passes
-> REFACTOR: clean only after green, without expanding behavior
-> Verify: rerun focused validation and then project validation when appropriate
```

Do not start the next behavior until the current cycle is green.

## RED Criteria

RED is valid only when all are true:

- The test was written before the production implementation for this behavior.
- The test fails.
- The failure message proves the target behavior is missing or wrong.
- The failure is not caused by syntax errors, broken imports, missing fixtures, bad test setup, timeouts, or environment misconfiguration.
- The assertion is strong enough that it would catch the bug or missing behavior.

If the test passes immediately, stop and explain whether the behavior already exists, the test is too weak, or the wrong path is under test.

If implementation already happened before a test, stop expanding the implementation. Add a characterization or regression test that would have failed against the old behavior, and clearly report that this is recovery from non-strict TDD.

## Test Quality Rules

Prefer tests that:

- exercise real production behavior through public or stable boundaries
- verify one behavior per test
- use clear names that state the behavior
- include realistic inputs and edge cases for the slice
- keep mocks at system boundaries such as network, time, filesystem, external services, or nondeterminism

Avoid tests that:

- only assert that mocks were called
- duplicate implementation details
- require test-only production APIs
- use weak assertions that would pass for the wrong behavior
- change tests only to make a failing implementation pass

When adding mocks, test-only seams, broad fixtures, or complex setup, read `references/testing-anti-patterns.md` before proceeding.

## Stop Conditions

Stop and report a blocker when any of these happen:

- No focused test command exists and no project rule explains an alternative.
- The test cannot be made to fail for the expected reason.
- The agent is about to write implementation before RED.
- Fixing the failing test requires changing the test's intended behavior.
- The slice grows beyond the plan, issue, or user request.
- Three consecutive attempts fail to reach GREEN.
- The project has no safe way to verify the behavior without user input or environment setup.

When stopped, give the smallest next decision needed from the user or the project docs.

## Validation Path

Prefer validation commands in this order:

1. the narrow test command for the new or changed test
2. the package or module test command covering the touched area
3. `./scripts/test`
4. `./scripts/check`
5. the command documented in `docs/testing.md`

Completion requires fresh validation evidence:

```text
Identify -> Run -> Read -> Verify -> Claim
```

Read the complete command output and exit code before claiming success.

## Output Format

Before implementation:

```text
tdd-ready: yes / no

Behavior under test:
-

Test plan:
- test file:
- expected RED:
- focused command:
- project validation:

Scope limits:
-

Blockers:
-
```

After a TDD cycle:

```text
tdd-cycle: complete / blocked

Behavior:
-

RED evidence:
- test:
- command:
- failure reason:

GREEN evidence:
- command:
- result:

Files changed:
-

Validation:
- focused:
- project:
- not run:

Residual risks:
-

Next gate:
- next TDD cycle / review-gate / final-check:
```

## Safety Rules

- Do not broaden the requested behavior while making tests pass.
- Do not do unrelated refactors, renames, formatting, dependency upgrades, or directory moves.
- Do not delete or rewrite user changes to regain strict TDD.
- Do not modify tests to match an incorrect implementation.
- Do not claim completion without fresh validation evidence.
