# 新项目 Harness 文件模板

这些模板用于从零初始化一个适合 coding agent 协作的新项目。它们是起点，不是必须一次性全部写满。

## 最小目录结构

```text
project/
├── .gitignore
├── AGENTS.md
├── README.md
├── docs/
│   ├── README.md
│   ├── roadmap.md
│   ├── project-status.md
│   ├── development.md
│   ├── testing.md
│   ├── architecture/
│   │   └── overview.md
│   ├── decisions/
│   │   └── README.md
│   ├── plans/
│   │   └── README.md
│   ├── specs/
│   │   └── README.md
│   └── troubleshooting.md
├── scripts/
│   ├── bootstrap
│   ├── check
│   ├── test
│   └── dev
└── src/
```

如果项目很小，至少保留：

```text
AGENTS.md
.gitignore
README.md
docs/README.md
docs/roadmap.md
docs/project-status.md
docs/development.md
docs/testing.md
scripts/check
```

## `.gitignore`

```gitignore
# OS / editor
.DS_Store
.idea/
.vscode/

# environment / secrets
.env
.env.*
!.env.example

# logs / caches
*.log
.cache/
.tmp/
tmp/

# coverage / reports
coverage/
.coverage

# build outputs
dist/
build/
out/
```

按技术栈补充：

| 技术栈 | 常见忽略项 |
|---|---|
| Node / TypeScript | `node_modules/`, `.next/`, `.nuxt/`, `vite.config.*.timestamp-*` |
| Rust | `target/` |
| Python | `.venv/`, `__pycache__/`, `.pytest_cache/`, `.ruff_cache/`, `*.pyc` |
| Go | `bin/`, `*.test`, `coverage.out` |
| Java / Kotlin | `.gradle/`, `build/`, `target/` |

## `README.md`

````md
# Project Name

一句话说明项目是什么。

## 目标

- 目标 1
- 目标 2
- 暂时不做什么

## 快速开始

```bash
./scripts/bootstrap
./scripts/dev
```

## 验证

```bash
./scripts/check
```
````

## `AGENTS.md`

```md
# AGENTS.md

这个项目使用 agent 协助开发，但 agent 必须遵守本文件约定。这里是入口地图，不是全部上下文。

## 必读顺序

1. `README.md`：项目目标、运行方式和用户视角。
2. `docs/README.md`：文档地图。
3. `docs/project-status.md`：当前进度、风险和下一步。
4. `docs/roadmap.md`：长期方向、Now/Next/Later 和暂时不做。
5. `docs/development.md`：开发命令、工具链和本地环境。
6. `docs/testing.md`：测试策略和验证命令。
7. 与当前任务相关的 `docs/architecture/`、`docs/decisions/`、`docs/specs/` 或 `docs/plans/` 文件。

## 工作方式

- 先理解需求和现有约定，再改代码。
- 对非平凡任务，先区分 Roadmap、Spec、Execution Plan 和 Project Status：长期方向放 roadmap，需求意图放 spec，执行步骤放 plans，当前接手点放 project-status。
- 如果需求、边界或验收标准不清楚，先写或更新 Spec，不要直接写实现计划。
- Execution Plan 必须包含范围、非目标、关联 Spec / Roadmap、影响文件、reviewable slices、测试、验证命令和回滚方式。
- 默认使用 TDD：先写失败测试，再写实现，再重构。
- 遇到 bug 时，先复现和定位根因，不允许猜测式修复。
- 完成前必须运行验证命令，并在回复中说明验证结果。
- 完成前必须检查 `git status` 和 `git diff`，确认没有无关改动、生成产物或 secret。
- 可以建议提交拆分和 commit message；不要直接执行 `git commit`，除非用户明确授权。

## 个人编程偏好

- 默认本地环境是 macOS + zsh，但项目命令必须通过 `scripts/*` 或包管理器脚本暴露，不依赖个人 shell alias 或绝对路径。
- 代码风格保持简洁、直接、高内聚、低耦合；优先沿用项目已有风格，不为了“看起来高级”新增抽象。
- 只实现当前需求需要的最小能力；不要顺手做 speculative feature、drive-by refactor 或无关格式化。
- 错误处理要显式；不要静默吞错，也不要把真实错误包装成无法排查的泛化信息。

## 修改边界

- 不要删除用户已有内容，除非用户明确要求。
- 不要运行破坏性 git 命令，例如 `git reset --hard`、`git checkout --`。
- 不要在未获授权时执行 `git commit`、`git tag`、`git push`。
- 不要随意引入新依赖；确需引入时先说明理由和替代方案。
- 不要扩大任务范围；发现范围问题时先记录并确认。
- 不要把 secret、token、私钥、生产数据写入代码、日志或文档。

## 完成标准

一次任务完成时，必须给出：

- 改了什么。
- 为什么这样改。
- 运行了哪些验证命令。
- 验证结果是什么。
- 还有哪些风险或后续事项。
```

## `docs/README.md`

```md
# Docs

这里存放项目长期上下文。不要一次性读取全部文档；根据任务按需读取。

## 推荐读取顺序

1. `project-status.md`：当前状态和下一步。
2. `roadmap.md`：长期方向、Now/Next/Later 和暂时不做。
3. `development.md`：本地开发、脚本、依赖、环境变量。
4. `testing.md`：测试策略、TDD 约定、验证命令。
5. `architecture/overview.md`：系统结构和边界。
6. `decisions/`：重要技术决策记录。
7. `specs/`：复杂功能的需求意图、验收标准和设计边界。
8. `plans/`：单个任务的 Execution Plan 和 slice 状态。
9. `troubleshooting.md`：常见问题和失败记录。

## 写入约定

- 稳定知识写入 docs。
- Roadmap 控制方向，Spec 控制意图，Execution Plan 控制行动，Project Status 控制接手。
- 临时推理留在对话或计划草稿，不要污染 roadmap。
- 已验证的失败和修复写入 `troubleshooting.md`。
- 影响架构的选择写入 `decisions/`。
```

## `docs/roadmap.md`

```md
# Roadmap

更新日期：YYYY-MM-DD

## 项目目标

-

## 当前阶段

-

## Now

| 事项 | 状态 | 为什么现在做 | 相关文档 |
|---|---|---|---|
|  | planned / active / blocked |  |  |

## Next

| 事项 | 触发条件 | 风险 | 相关文档 |
|---|---|---|---|
|  |  |  |  |

## Later

| 事项 | 暂缓原因 |
|---|---|
|  |  |

## 暂时不做

| 事项 | 不做原因 | 何时重新评估 |
|---|---|---|
|  |  |  |

## 当前 Active Work

- Active spec：
- Active plan：
- 当前状态：
```

## `docs/project-status.md`

```md
# Project Status

更新日期：YYYY-MM-DD

## 当前目标

-

## Active Work

- Roadmap item：
- Spec：
- Plan：
- 当前 slice：

## 已完成

-

## 进行中

-

## 下一步

-

## 需要人类决定

-

## 风险和未决问题

-

## 最近验证

| 日期 | 命令 | 结果 | 备注 |
|---|---|---|---|
```

## `docs/development.md`

````md
# Development

## 环境要求

- 本人日常环境：macOS + zsh。
- 项目脚本不要依赖个人 shell alias、绝对路径或本机私有配置。
- Runtime：
- Package manager：
- Database：
- External services：

## 初始化

```bash
./scripts/bootstrap
```

## 本地开发

```bash
./scripts/dev
```

## 常用命令

| 任务 | 命令 |
|---|---|
| 安装依赖 | `./scripts/bootstrap` |
| 启动开发 | `./scripts/dev` |
| 运行测试 | `./scripts/test` |
| 完整检查 | `./scripts/check` |

## Git 工作流

- 项目初始化时创建 Git 仓库和 `.gitignore`。
- 开发中发现可重复生成的产物进入 `git status`，先更新 `.gitignore`。
- 每个非平凡任务完成前运行 `git status` 和 `git diff`。
- 提交按 reviewable slice 拆分，不把无关修改混在一个 commit。
- 默认由 agent 建议提交拆分和 commit message；只有用户明确授权时才执行 `git commit`。

## Commit Message

使用 Conventional Commits：

```text
type(scope): summary
```

## 环境变量

使用 `.env.example` 记录变量名和用途，不要提交真实 secret。

| 变量 | 必需 | 用途 |
|---|---|---|
````

## `docs/testing.md`

````md
# Testing

## 测试策略

- 单元测试覆盖纯逻辑、边界条件和错误处理。
- 集成测试覆盖模块交互、数据库、外部服务适配层。
- E2E 测试覆盖关键用户路径。
- 回归 bug 必须先写能复现 bug 的失败测试。
- 测试名称描述行为；如果语言允许，使用 `snake_case`。
- 测试 fixture 放在 `tests/fixtures/` 或当前生态的标准 fixture 目录。

## TDD 流程

1. RED：写一个最小失败测试。
2. Verify RED：运行测试，确认它因为预期原因失败。
3. GREEN：写最小实现让测试通过。
4. Verify GREEN：运行测试，确认通过。
5. REFACTOR：在测试保护下清理结构。
6. Final Check：运行完整验证命令。

## 验证命令

```bash
./scripts/test
./scripts/check
```

## 完成声明规则

没有新的验证结果，不允许说“完成”“修好了”“测试通过”。
````

## `docs/architecture/overview.md`

````md
# Architecture Overview

## 模块边界

| 模块 | 职责 | 不负责 |
|---|---|---|

## 数据流

```text
input -> domain -> persistence -> output
```

## 关键约定

-

## 禁止事项

-
````

## `docs/decisions/0001-choose-technology-stack.md`

```md
# 0001 Choose Technology Stack

日期：YYYY-MM-DD

## 决策

本项目选择：

## 背景

## 备选方案

## 选择理由

## 代价和风险

## 何时重新评估
```

## `docs/specs/YYYY-MM-DD-feature-name.md`

```md
# Spec: Feature Name

日期：YYYY-MM-DD

## 背景

## 目标

## 非目标

## 用户场景

## 验收标准

## 当前上下文

## 设计方案

## 备选方案

## 风险和边界

## 测试策略

## Open Questions
```

## `docs/plans/YYYY-MM-DD-feature-name.md`

````md
# Plan: Feature Name

日期：YYYY-MM-DD

## 目标

## 非目标

## 关联文档

- Roadmap：
- Spec：
- ADR：

## Spec 摘要

如果没有单独的 `docs/specs/*.md`，这里写清背景、验收标准和设计边界。

## Blast Radius

| 维度 | 是否影响 | 说明 |
|---|---|---|
| 公共 API |  |  |
| 数据模型 |  |  |
| 权限 / 安全 |  |  |
| 网络 / 外部服务 |  |  |
| 部署 / 配置 |  |  |
| 测试夹具 |  |  |

## 影响文件

| 文件 | 预期改动 |
|---|---|

## 验收标准

- [ ]

## Reviewable Slices

### Slice 1：名称

状态：todo / doing / done / blocked

目标：

测试：

步骤：

- [ ] 写失败测试：
- [ ] 运行局部测试，确认因为预期原因失败：
- [ ] 写最小实现：
- [ ] 运行局部测试，确认通过：
- [ ] 必要重构：
- [ ] 运行验证命令：
- [ ] 更新计划状态：

验证命令：

```bash

```

回滚方式：

## 风险

## 完成标准

## 计划变更记录
````
