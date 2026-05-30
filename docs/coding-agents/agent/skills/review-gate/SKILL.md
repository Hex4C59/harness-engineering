---
name: review-gate
description: Use when a completed coding slice, bug fix, refactor, or documentation/tooling change needs review before commit. Trigger when the user says "review gate", asks for a reviewer agent, asks whether a diff is safe/correct, finishes implementation before commit-gate, or changes public APIs, data models, auth/security, concurrency, deployment, tests, or multiple files/modules. This skill is read-only by default: do not modify files, stage changes, commit, tag, or push unless the user explicitly authorizes that exact action.
---

# Review Gate

Use this skill to decide whether the current diff is correct, scoped, tested, and safe enough to proceed toward commit.

Review Gate protects quality. Commit Gate protects project history. Run Review Gate before Commit Gate when both apply.

## Read Project Rules

Before judging, read these if present:

1. `AGENTS.md`
2. `docs/agent/playbooks/workflows/review-and-finish.md`
3. `docs/agent/playbooks/principles/reviewable-slices.md`
4. `docs/agent/playbooks/checklists/review-and-finish.md`
5. `docs/development.md`
6. `docs/testing.md`
7. `docs/project-status.md`

Project-local rules override this skill when they are stricter.

## When to Review

Use a review pass when any of these are true:

- The user asks for review, review gate, reviewer agent, independent review, or "is this safe/correct?".
- A reviewable slice is implemented and the next step would be commit-gate.
- The diff touches multiple files or modules.
- The diff changes public APIs, data models, auth, security, permissions, concurrency, networking, storage, deployment, CI, or build tooling.
- The task fixed a bug with non-trivial root cause.
- The implementation was produced by an agent from a plan and needs plan-vs-diff validation.
- The user or agent feels the diff "looks plausible" but has not been independently checked.
- The commit boundary is unclear or scope creep is suspected.

You may skip a dedicated review pass for tiny changes when all are true:

- typo, formatting-only, or tiny docs-only edit
- single-line config or metadata change
- no behavior, API, security, deployment, or dependency impact
- validation is already fresh and proportionate to the change
- the user did not ask for review

If uncertain, do a lightweight read-only review.

## Inspect State

Run and read:

```bash
git status --short
git diff --stat
git diff
git diff --cached --stat
git diff --cached
```

If untracked files exist, inspect their names and purpose before judging scope. Do not stage files.

If there is a plan, spec, issue, bug report, or user request in the current conversation or project docs, compare the diff against it.

## Preferred Review Modes

Choose the strongest available review mode that fits the environment.

1. **Independent reviewer agent / subagent**: If the runtime supports subagents, delegate a read-only review to a fresh reviewer context with the task goal, relevant plan/spec, diff range, and `references/reviewer-agent.md`.
2. **Same-session fallback**: If subagents are unavailable, read `references/reviewer-agent.md`, perform the read-only review in the current session, and explicitly state that it was not an independent-context review.

Do not use a reviewer pass as permission to modify files. Fixes require a separate implementation step after findings are accepted.

## Reviewer Instructions

When delegating to a reviewer agent or doing same-session fallback, load `references/reviewer-agent.md` and follow it exactly. It defines the read-only reviewer role, allowed finding categories, severity meanings, output requirements, and style-preference exclusions.

Pass only task-local context to the reviewer: user request, plan/spec if present, project rules that matter, diff range or current diff, and validation already run. Do not pass your implementation rationale as ground truth.

## Review-Ready Criteria

Return `review-ready: yes` only when all are true:

- The diff matches the requested task, plan, or reviewable slice.
- No blocking correctness, security, compatibility, or scope issue is found.
- Tests or validation are fresh enough for the risk level, or missing validation is explicitly accepted by the user.
- The diff is small enough to review or has a clear split plan.
- There are no unrelated generated outputs, caches, secrets, debug prints, temporary files, or machine-specific paths.
- Required docs/status/plan updates are present when the change affects documented behavior.

If any criterion fails, return `review-ready: no` and list blockers.

## Relationship to Commit Gate

Use this sequence:

```text
implementation complete
-> run relevant validation
-> review-gate
-> fix accepted blocking / important findings
-> rerun relevant validation
-> commit-gate
-> user-authorized git add / git commit only if requested
```

Do not report `commit-ready: yes`; that is Commit Gate's job.

If Review Gate finds blockers, Commit Gate should not be marked ready until blockers are fixed or explicitly accepted by the user.

## Output Format

If ready:

```text
review-ready: yes

Review mode:
- independent reviewer agent / same-session fallback:

Validation considered:
- command:
- result:

Findings:
- none blocking

Residual risks:
-

Next gate:
- commit-gate recommended / not needed because:
```

If not ready:

```text
review-ready: no

Review mode:
- independent reviewer agent / same-session fallback:

Findings:
1. severity: blocking / important
   file:
   issue:
   why it matters:
   suggested fix:

Validation gaps:
-

Next steps:
-

Commit gate:
- do not run / not ready until:
```

## Safety Rules

- Read-only by default.
- Do not modify files while performing the review.
- Do not stage, commit, tag, or push.
- Do not run destructive commands.
- Do not report style-only opinions as findings.
- Do not trust an agent's implementation summary without checking the actual diff.
- If unrelated user changes exist, call them out and keep them out of review conclusions unless they affect the current task.
