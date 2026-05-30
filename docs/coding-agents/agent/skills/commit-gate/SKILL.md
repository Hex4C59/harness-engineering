---
name: commit-gate
description: Use when deciding whether the current git worktree is ready to commit, how to split commits, and what commit messages to use. Trigger when the user asks whether work can be committed, asks for commit messages, says "commit gate", finishes a reviewable slice, or needs commit boundary review. Do not run git commit, git add, git tag, or git push unless the user explicitly authorizes that exact action.
---

# Commit Gate

Use this skill to decide whether the current worktree has reached a clean commit boundary.

The job is to protect project history, not to rush code into Git.

## Read Project Rules

Before judging, read these if present:

1. `AGENTS.md`
2. `docs/agent/playbooks/workflows/review-and-finish.md`
3. `docs/agent/playbooks/prompts/commit-gate.md`
4. `docs/development.md`
5. `docs/project-status.md`

Project-local rules override this skill when they are stricter.

## Inspect Git State

Run and read:

```bash
git status --short
git status --ignored --short
git diff --stat
git diff
git diff --cached --stat
git diff --cached
```

If the repository has no commits yet, also note that this is an initial commit candidate.

Classify changes as:

- `staged`
- `unstaged`
- `untracked`
- `ignored`
- `unrelated to this task`

Do not assume staged files are the correct commit set. Review staged and unstaged changes separately.

## Validation Gate

A worktree is not commit-ready without fresh validation evidence.

Fresh validation means one of:

- The current conversation contains a recent successful validation command after the latest relevant changes.
- `docs/project-status.md` records a recent successful validation for this exact slice.
- You run the project validation command now.

Prefer project validation commands in this order:

1. `./scripts/check`
2. the command documented in `docs/testing.md`
3. the package manager's test/check script

If validation cannot be run, set `commit-ready: no` unless the user explicitly wants a commit plan with known unverified risk.

## Commit-Ready Criteria

Return `commit-ready: yes` only when all are true:

- The completed work has a clear task or reviewable-slice boundary.
- Validation is fresh and passing, or the user explicitly accepts an unverified commit plan.
- The diff is reviewable and scoped to the task.
- Required docs/status files are updated.
- There are no unrelated changes mixed into the recommended commit.
- There are no generated outputs, dependency caches, local toolboxes, secrets, credentials, debug prints, temporary files, or machine-specific paths in the recommended commit.
- The commit can be explained as one coherent project-history step.

If any criterion fails, return `commit-ready: no` and list blockers.

## Commit Boundary Rules

Use these default boundaries:

| Change type | Commit boundary |
|---|---|
| New project initialization | Harness, scripts, docs, base `.gitignore`, and initial toolchain can be one initial commit if they were completed together. |
| New feature | One completed reviewable slice per commit. |
| Bug fix | Reproduction test and minimal fix in the same commit. |
| Refactor | Behavior-preserving refactor in its own commit. |
| Docs | Docs-only changes in their own commit. |
| Dependencies/tooling | Dependency, lockfile, and tool configuration changes in their own commit. |
| Formatting | Broad formatting changes stay out of feature commits. |

If the worktree contains multiple logical changes, recommend multiple commits.

## Exclude By Default

Never include these unless the user explicitly asks and the project rules allow it:

- ignored files
- `node_modules/`, `.venv/`, `target/`, `dist/`, `build/`, coverage reports, caches
- `.env`, secrets, tokens, private keys, production data
- `docs/agent/playbooks/` when it is a local copied toolbox
- real MCP credentials or personal MCP config
- temporary experiments, debug logs, screenshots, downloaded files
- unrelated user changes

## Commit Message Style

Use Conventional Commits:

```text
type(scope): summary
```

Default to Chinese summaries while keeping the Conventional Commit type in English.

Examples:

```text
chore(agent): 初始化项目 harness
feat(cli): 添加最小问候命令
fix(parser): 修复空输入处理
docs(agent): 说明 playbooks 跟踪策略
test(cli): 补充命令输出回归测试
```

Use `chore`, `feat`, `fix`, `docs`, `test`, `refactor`, `build`, or `ci` unless project rules define another type set.

## Output Format

If ready:

```text
commit-ready: yes

Validation:
- command:
- result:

Recommended commits:
1. type(scope): summary
   Include:
   - path
   Exclude:
   - path
   Why this boundary:
   -

Notes:
-
```

If not ready:

```text
commit-ready: no

Blockers:
-

Next steps:
-

Possible commit boundary after blockers are fixed:
-
```

## Safety Rules

- Do not run `git add`, `git commit`, `git tag`, or `git push` unless the user explicitly authorizes that exact action.
- Do not stage files as part of analysis.
- Do not modify files unless the user asks you to fix blockers.
- If the user authorizes committing, restate the exact files and commit message before running the command.
- If there are unrelated changes, keep them out of the recommended commit plan and call them out.
