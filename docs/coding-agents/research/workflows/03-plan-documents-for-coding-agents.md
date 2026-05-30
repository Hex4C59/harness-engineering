# Coding Agent 计划文档分层：Roadmap、Spec、Execution Plan 和 Status

调研日期：2026-05-30

## 这篇文档回答什么

你提出了一个很关键的问题：

```text
计划文档是否应该分成两种？
一种是 roadmap 这种长期路线和状态表。
一种是未完成 slice / task 的执行清单。
```

这个判断方向是对的，但调研之后更准确的拆法不是两层，而是四层：

1. `Roadmap`：长期方向、优先级和状态。
2. `Spec / Design`：某个功能到底要什么、不要什么、如何验收。
3. `Execution Plan`：当前 slice / task 如何按 TDD 落地。
4. `Progress Status`：当前做到哪里，下一步是什么，谁能接手。

核心结论：

> 不要把所有东西都叫 plan。Roadmap 决定“值得往哪里走”；Spec 定义“走到哪里才算到”；Execution Plan 规定“这一小段怎么走”；Progress Status 记录“现在走到哪了”。

这件事在 coding agent 工作流里尤其重要，因为 agent 最大的问题往往不是不会写代码，而是：

- 目标和非目标不清楚。
- 计划和状态混在一起。
- 执行时重新设计一遍。
- 一次做太多，diff 难以 review。
- 测试和验收没有变成外部约束。
- 新会话接手时不知道当前真实状态。

好的计划文档不是为了“显得专业”，而是为了把 agent 的自由度压缩在可验证、可回滚、可 review 的范围内。

## 来源说明

本文主要参考公开博客、官方文档、项目源码和研究报告。涉及具体工具能力的描述，以 2026-05-30 可查公开资料为准。

| 来源 | 类型 | 主要价值 |
|---|---|---|
| [Best practices for Claude Code](https://code.claude.com/docs/en/best-practices) | Anthropic 官方文档 | 强调 `Explore -> Plan -> Implement -> Commit`，先理解代码，再计划，再实现，并给 agent 可验证标准 |
| [How I'm using coding agents in September, 2025](https://blog.fsck.com/2025/10/05/how-im-using-coding-agents-in-september-2025) | Jesse Vincent 实践博客 | architect / implementer 双会话、计划文档、分批执行、回到 architect review |
| [Superpowers: How I'm using coding agents in October 2025](https://blog.fsck.com/2025/10/09/superpowers) | Jesse Vincent 实践博客 | 把 brainstorm、plan、TDD、review、subagent 变成 skills 和 workflow gates |
| [superpowers writing-plans SKILL.md](https://github.com/obra/superpowers/blob/main/skills/writing-plans/SKILL.md) | Skill 源码 | 把计划写成精确、可执行、可验证的微任务脚本 |
| [My LLM codegen workflow atm](https://harper.blog/2025/02/16/my-llm-codegen-workflow-atm/) | Harper Reed 实践博客 | `spec.md`、`prompt_plan.md`、`todo.md` 三段式 LLM codegen 工作流 |
| [Simon Willison 对 Harper Reed 工作流的解读](https://simonwillison.net/2025/Feb/21/my-llm-codegen-workflow-atm/) | 资深开发者评论 | 强调 spec、prompt plan、todo 作为多轮模型调用之间的状态载体 |
| [Spec-driven development with AI](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/) | GitHub 官方博客 | Spec Kit 的 `specify -> plan -> tasks -> implement` 流程 |
| [Diving Into Spec-Driven Development With GitHub Spec Kit](https://developer.microsoft.com/blog/spec-driven-development-spec-kit) | Microsoft 开发者博客 | 解释 `/specify`、`/plan`、`/tasks` 的职责边界 |
| [Magentic-One](https://www.microsoft.com/en-us/research/articles/magentic-one-a-generalist-multi-agent-system-for-solving-complex-tasks/) | Microsoft Research | `Task Ledger` 和 `Progress Ledger` 的双账本思想 |
| [Magentic-One AutoGen 文档](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/magentic-one.html) | 官方项目文档 | Orchestrator 如何计划、跟踪进度、发现停滞后 re-plan |
| [Shape Up](https://basecamp.com/shapeup) | Basecamp 产品开发方法 | Roadmap 不应退化成无限 backlog；应关注 appetite、边界、vertical slices |
| [Agentic Engineering Patterns](https://simonwillison.net/guides/agentic-engineering-patterns/) | Simon Willison 方法论 | 把 coding agent 使用方式抽象成工程模式，强调测试、Git、可 review 工作流 |

## 事实、推断和个人整理框架

为了避免把不同层次的东西混成“结论”，本文区分三类内容：

| 类型 | 本文含义 |
|---|---|
| 事实 | 来源中明确出现的流程、文档名、实践或工具设计 |
| 推断 | 从多个来源共同模式中归纳出的工程含义 |
| 个人整理框架 | 为你的 `AGENTS.md + docs + TDD + Codex` 工作流整理出的推荐落地方式 |

例如：

- Jesse Vincent 确实使用 architect / implementer 分工，这是事实。
- 这说明“计划上下文和执行上下文应该分离”，这是推断。
- 建议你在个人项目里维护 `docs/roadmap.md`、`docs/project-status.md`、`docs/plans/*.md`，这是个人整理框架。

## 为什么“计划”容易混乱

中文里我们经常把下面几种东西都叫“计划”：

- 项目接下来半年想做什么。
- 这个功能为什么值得做。
- 这个功能应该怎么设计。
- 当前 PR 要改哪些文件。
- 今天 agent 要执行哪个 slice。
- 现在做到哪一步。
- 还有哪些 TODO 没完成。

但这些东西的生命周期完全不同。

Roadmap 可能几周或几个月更新一次。Execution Plan 可能一个下午就失效。Project Status 可能每个 slice 后都要更新。Spec 在需求变化时更新，但不应该被实现过程里的琐碎状态污染。

把它们混在一个文件里，会带来几个典型问题：

1. 长期方向被细碎 TODO 淹没。
2. 当前状态看起来像长期承诺。
3. agent 执行时拿 roadmap 当任务清单。
4. 人类 review 时不知道当前 diff 对应哪个验收标准。
5. 新会话接手时读到一堆过期任务。
6. 计划越写越长，最后没人再信它。

所以真正的问题不是“要不要写计划”，而是：

```text
这个文档到底给谁读？
它回答哪个时间尺度的问题？
它什么时候更新？
它过期后谁负责处理？
```

## 四层模型

### 第一层：Roadmap

Roadmap 回答：

```text
接下来什么方向值得做？
哪些事情正在做？
哪些事情稍后再做？
哪些事情明确不做？
为什么这样排序？
```

它的读者主要是人类，以及刚恢复上下文的 agent。它不应该指导 agent 直接改代码。

推荐内容：

- 项目目标。
- 当前阶段。
- `Now / Next / Later`。
- 每个方向的状态。
- 重要依赖和阻塞。
- 最近一次复盘日期。
- 当前 active spec / active plan 链接。
- 暂时不做的方向。

不推荐放进 Roadmap：

- 逐文件修改步骤。
- 具体测试命令。
- 过细的 TODO。
- 每个函数怎么实现。
- 每次执行后的日志。
- 已完成 slice 的冗长流水账。

一个好的 Roadmap 更像方向盘和仪表盘，不是螺丝刀。

示例模板：

```md
# Roadmap

更新日期：YYYY-MM-DD

## 项目目标

## 当前阶段

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

### 第二层：Spec / Design

Spec 或 Design 回答：

```text
这个功能到底要解决什么问题？
成功标准是什么？
哪些边界不做？
有哪些设计选择？
有哪些测试和验收标准？
```

它的读者是人类、architect agent、reviewer agent 和 implementer agent。它不是执行清单，但它是执行清单的上游。

一个可用的 Spec 至少应该包含：

- 背景。
- 目标。
- 非目标。
- 用户场景或调用场景。
- 验收标准。
- 当前系统上下文。
- 设计方案。
- 备选方案和取舍。
- 数据、API、权限、部署、兼容性影响。
- 边界条件。
- 测试策略。
- Open questions。

什么时候需要单独 Spec？

| 任务类型 | 是否需要单独 Spec |
|---|---|
| 改 typo、改文案、改配置 | 不需要 |
| 小 bug，有明确复现 | 通常不需要，直接写回归测试和短计划 |
| 中等功能，影响多个文件 | 可以把 Spec 和 Plan 合在一个 `docs/plans/*.md` |
| 架构调整、迁移、跨模块变化 | 建议单独 Spec |
| 需求不清楚 | 先写 Spec，不要写实现计划 |
| 多人协作或长期功能 | 建议单独 Spec |

推荐模板：

```md
# Feature Spec: 功能名

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

Spec 最重要的作用是防止 agent 把“能实现”误认为“该实现”。它定义的是意图和边界，而不是步骤。

### 第三层：Execution Plan

Execution Plan 回答：

```text
当前这个 feature / bugfix / refactor 要分成哪些 reviewable slices？
每个 slice 改哪些文件？
先写什么失败测试？
运行哪些命令？
完成证据是什么？
如果出错怎么回滚？
```

它的读者主要是 implementer agent。和 Roadmap 不同，它可以非常具体，甚至刻意具体。

Superpowers 的 `writing-plans` 对这一点非常极端：计划应该写给“上下文很少、判断力有限、容易跳过测试的执行者”。每个任务都要有精确路径、具体代码、命令和预期输出，并且禁止 `TODO`、`TBD`、空泛的“处理边界情况”。

不一定所有个人项目都要做到这个粒度，但它给了一个重要原则：

> Execution Plan 不是愿望清单，而是执行协议。

推荐内容：

- 计划目标。
- 非目标。
- 关联 Spec / Roadmap item。
- Blast radius。
- 影响文件。
- Reviewable slices。
- 每个 slice 的测试方式。
- 每个 slice 的回滚方式。
- TDD 顺序。
- 局部验证命令。
- 完整验证命令。
- 当前任务状态。

推荐模板：

````md
# 功能名 Implementation Plan

日期：YYYY-MM-DD

## 目标

## 非目标

## 关联文档

- Spec：
- Roadmap：
- ADR：

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
|  |  |

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

### Slice 2：名称

状态：todo

目标：

测试：

验证命令：

回滚方式：

## 完成标准

## 计划变更记录
````

Execution Plan 的关键不是“写得长”，而是让 agent 没有理由跳过下面这些动作：

1. 先写失败测试。
2. 只实现当前 slice。
3. 只改计划内文件。
4. 跑计划里的验证命令。
5. 完成后更新状态。
6. 发现计划错了就停下来，而不是擅自扩大范围。

### 第四层：Progress Status

Progress Status 回答：

```text
现在真实做到哪里？
最近一次验证是什么结果？
当前 active plan 是哪个？
下一个建议动作是什么？
有没有阻塞？
新会话应该从哪里继续？
```

它的读者是未来的你、新开的 Codex 会话、reviewer agent 和接手者。

Magentic-One 的 `Task Ledger` 和 `Progress Ledger` 很适合作为类比：

- `Task Ledger` 保存任务事实、假设和整体计划。
- `Progress Ledger` 保存当前进度、下一步、是否停滞、是否需要 re-plan。

对个人项目来说，`docs/project-status.md` 就是 Progress Ledger。

推荐内容：

- 当前项目目标。
- 当前 active work。
- 已完成事项。
- 进行中事项。
- 当前阻塞。
- 最近验证命令和结果。
- 下一个建议 slice。
- 最近更新日期。
- 需要人类决定的问题。

推荐模板：

```md
# Project Status

更新日期：YYYY-MM-DD

## 当前目标

## Active Work

- Roadmap item：
- Spec：
- Plan：
- 当前 slice：

## 已完成

## 进行中

## 阻塞

## 最近验证

| 时间 | 命令 | 结果 | 说明 |
|---|---|---|---|
|  |  |  |  |

## 下一步

## 需要人类决定
```

Progress Status 的质量直接决定新会话能不能接手。如果它过期，agent 会自然开始猜。

## 各来源怎么做计划

### Anthropic Claude Code：Explore -> Plan -> Implement -> Commit

Anthropic 的 Claude Code best practices 明确给出一个阶段性流程：

```text
Explore -> Plan -> Implement -> Commit
```

这个流程的重点不是形式，而是顺序：

1. 先让 agent 阅读相关代码和约定。
2. 再问它需要改哪些文件、session flow 是什么、计划是什么。
3. 再让它按计划实现。
4. 实现时写测试、运行测试、修复失败。
5. 最后提交或开 PR。

对本文的启发：

- `Explore` 不能省，否则计划是在猜。
- `Plan` 应该先于写代码。
- `Implement` 应该引用计划，而不是重新规划。
- `Commit` 或 PR 应该建立在验证结果上。

这支持把计划文档分层：探索结果和长期背景不应该混进执行清单；执行清单必须能连接到验证命令。

### Jesse Vincent：architect / implementer 双会话

Jesse Vincent 的流程里，一个会话扮演 architect，负责 brainstorm、设计和实施计划；另一个新会话扮演 implementer，只读设计文档和计划文档，然后执行前几个任务。

实现完成后，他会回到 architect 会话，让它仔细 review。确认后，再让 implementer 更新 planning doc 的当前状态，然后清空 implementer 会话，从下一批任务继续。

这个流程有几个关键点：

1. 计划上下文和实现上下文分离。
2. 实现者不能偏离计划。
3. 每次只执行一批任务。
4. review 使用相对新鲜的上下文。
5. planning doc 是状态载体，不只是一次性文本。

对个人 Codex 工作流的启发：

```text
一个会话写计划。
另一个会话只执行当前 slice。
第三个视角或同一会话的 review gate 审 diff。
完成后更新 plan 和 project-status。
```

如果你只用一个 Codex 会话，也可以模拟这个分工：

- 先要求“只写计划，不改代码”。
- 计划确认后，要求“只执行第一个未完成 slice”。
- 实现后，要求“按原计划 review 当前 diff”。

### Superpowers：计划是给 agent 执行的脚本

Superpowers 的 `writing-plans` 是最适合研究 Execution Plan 的材料。

它要求计划包含：

- 精确文件路径。
- 创建或修改哪些文件。
- 实际代码片段。
- 运行什么命令。
- 预期输出是什么。
- 何时提交。

它禁止：

- `TODO` / `TBD`。
- “添加适当错误处理”这种空话。
- “写上面的测试”但不给测试细节。
- “类似 Task N”。
- 引用尚未定义的类型、函数或方法。

这背后的判断是：

> agent 不是缺少写代码能力，而是缺少稳定边界、上下文记忆和工程纪律。

所以 Execution Plan 的作用是压缩 agent 的自由度。它不是给资深人类看的高层说明，而是给执行者的约束协议。

不过个人项目不一定要完全复制 Superpowers 的超细粒度。更合理的折中是：

- 高风险任务：写到文件、测试、命令、预期输出。
- 中风险任务：写 reviewable slices、影响文件和验证命令。
- 低风险任务：短计划或直接改，但完成前仍要验证。

### Harper Reed：spec.md、prompt_plan.md、todo.md

Harper Reed 的 LLM codegen 工作流很清楚地区分了几种 artifact：

- 先通过问答形成详细 `spec.md`。
- 再用 reasoning model 生成 `prompt_plan.md`。
- 再生成更低层的 `todo.md`。
- 代码编辑模型按 todo 逐步执行，并在多轮调用之间保持状态。

这个流程对个人工作区很有启发：

1. `spec.md` 保存意图，不保存执行流水账。
2. `prompt_plan.md` 保存给模型的分阶段实现入口。
3. `todo.md` 保存短周期状态。

如果映射到你的目录结构，可以是：

```text
docs/specs/YYYY-MM-DD-feature.md        可选，复杂功能使用
docs/plans/YYYY-MM-DD-feature.md        执行计划和 slices
docs/project-status.md                  当前状态和下一步
```

小项目可以把 Spec 和 Execution Plan 合并在一个 `docs/plans/*.md` 里，但要用章节区分：

```text
上半部分：目标、非目标、验收标准、设计选择。
下半部分：reviewable slices、TDD 步骤、验证命令、状态。
```

### GitHub Spec Kit：specify -> plan -> tasks -> implement

Spec Kit 的流程很适合作为命名参考：

```text
/specify -> /plan -> /tasks -> implement
```

这四步的职责大致是：

- `/specify`：定义 what 和 why。
- `/plan`：定义 technical how。
- `/tasks`：拆成可执行、可测试的小块。
- `implement`：按任务执行。

这说明在 agent 时代，Spec 不是写完就放一边的文档，而是驱动 plan、tasks 和 implementation 的中心 artifact。

对你的工作流的启发：

- Roadmap 不应该直接进入实现。
- Spec 应该先约束计划。
- Plan 应该连接技术路径。
- Tasks / slices 应该足够小，可以测试和 review。
- 每个阶段都应该有人类 checkpoint。

### Magentic-One：Task Ledger 和 Progress Ledger

Magentic-One 的 Orchestrator 使用两个账本：

- `Task Ledger`：保存任务计划、事实、假设和整体策略。
- `Progress Ledger`：保存当前进度、是否完成、下一步任务、是否需要调整。

这对个人 coding agent 工作流特别有价值，因为它解释了为什么“计划”和“状态”不能混在一起。

一个任务可能有稳定的总体目标，但当前进度会不断变化。把它们写在同一个长 checklist 里，很容易出现：

- 总体目标被频繁编辑。
- 进度状态失真。
- 已完成任务和未来计划混在一起。
- 新会话不知道哪些内容仍然有效。

更好的做法是：

```text
docs/plans/*.md          保存任务计划和 slice 状态
docs/project-status.md   保存全局当前状态和下一步
```

当 Progress Status 显示多次停滞，才回头更新 Execution Plan 或 Spec。

### Shape Up：Roadmap 不是无限 backlog

Shape Up 不是专门给 coding agent 的方法，但它对 Roadmap 很有启发。

它强调：

- 先 shaping，再交给团队 build。
- 不用估算替代判断，而是问 appetite：这件事值得花多少时间。
- shaped work 要有边界，明确不做什么。
- 团队在 build 过程中发现 tasks，而不是由管理者提前切成所有小票。
- 用 vertical slices 推进，而不是沉迷横向铺底。

对本文的启发：

1. Roadmap 不应该是无限 TODO 列表。
2. Roadmap item 应该有边界和暂不做。
3. 长期方向不应该过早拆成细碎任务。
4. 任务清单应该在进入具体 Execution Plan 后才展开。
5. 如果任务发现越来越多，不一定是坏事，但需要用 status 和 scope 控制。

这能修正一种常见误区：

```text
Roadmap 不是“所有想做的东西”。
Roadmap 是“在当前判断下，值得排队或下注的方向”。
```

## 两种计划的更准确说法

你原先的二分可以保留，但需要改名：

| 你的说法 | 更准确的说法 | 为什么 |
|---|---|---|
| roadmap 这种长期路线和状态表 | Roadmap + Project Status | 长期方向和当前状态更新频率不同，最好拆开 |
| 未完成 slice / task 的执行清单 | Execution Plan | 它不是长期计划，而是当前任务的执行协议 |

也就是说，最小可用结构是三份：

```text
docs/roadmap.md
docs/project-status.md
docs/plans/YYYY-MM-DD-feature-name.md
```

复杂项目再加：

```text
docs/specs/YYYY-MM-DD-feature-name.md
docs/decisions/NNNN-topic.md
```

## 不同任务该写什么

| 任务类型 | 推荐文档 |
|---|---|
| 改 typo、改一个配置 | 不写计划，直接改，完成前验证 |
| 小 bug，有明确复现 | 短 bugfix plan，或直接在对话里列 TDD 步骤 |
| 中等功能，影响多个文件 | `docs/plans/*.md`，包含目标、非目标、slices、测试、验证 |
| 大功能，需求仍会讨论 | 先 `docs/specs/*.md`，确认后再写 `docs/plans/*.md` |
| 架构调整或迁移 | Spec + ADR + Execution Plan |
| 长期产品方向 | `docs/roadmap.md` |
| 新会话接手 | `docs/project-status.md` |
| agent 执行一半中断 | 更新 `docs/project-status.md` 和当前 plan 的 slice 状态 |
| 发现计划方向错 | 停止实现，更新 Spec 或 Plan，不要边做边暗改 |

## 推荐目录结构

适合个人项目的结构：

```text
docs/
  README.md
  roadmap.md
  project-status.md
  development.md
  testing.md
  architecture/
    overview.md
  decisions/
    README.md
    0001-choose-technology-stack.md
  plans/
    README.md
    2026-05-30-feature-name.md
  specs/
    README.md
    2026-05-30-feature-name.md
  troubleshooting.md
```

如果项目还很小，可以先不建 `docs/specs/`：

```text
docs/
  project-status.md
  roadmap.md
  plans/
```

并在 `docs/plans/*.md` 里使用两个大节：

```text
## Spec
目标、非目标、验收标准、设计选择。

## Execution Plan
slices、TDD 步骤、验证命令、状态。
```

## 文档生命周期

推荐生命周期：

```text
idea
-> roadmap item
-> spec / design
-> execution plan
-> slice implementation
-> verification
-> project-status update
-> ADR / troubleshooting / docs update
```

每一步的责任不同：

| 阶段 | 主要产物 | 判断问题 |
|---|---|---|
| idea | 临时笔记或对话 | 值不值得继续想 |
| roadmap item | `docs/roadmap.md` | 值不值得排队 |
| spec | `docs/specs/*.md` 或 plan 的 Spec 章节 | 做什么，不做什么 |
| execution plan | `docs/plans/*.md` | 怎么拆 slice，怎么验证 |
| implementation | 代码和测试 | 当前 slice 是否完成 |
| verification | 测试输出、check 输出 | 有没有外部证据 |
| status update | `docs/project-status.md` | 新会话能否接手 |
| decision capture | `docs/decisions/*.md` | 哪些选择长期有效 |

## 如何判断计划是否足够好

### Roadmap 的好坏标准

好的 Roadmap：

- 能看出当前阶段。
- 能看出现在做什么、下一步做什么、暂时不做什么。
- 每个方向都有原因。
- 能链接到 active spec 或 active plan。
- 不把所有灵感都塞进 Now。
- 不把长期方向写成细碎任务。

坏的 Roadmap：

- 像一个无穷 backlog。
- 没有优先级。
- 没有暂不做。
- 没有更新时间。
- 和当前真实状态不一致。

### Spec 的好坏标准

好的 Spec：

- 读完能判断这个功能该不该做。
- 有目标和非目标。
- 有验收标准。
- 有关键边界条件。
- 有设计取舍。
- 有测试策略。
- 能指导 review。

坏的 Spec：

- 只有“实现某某功能”。
- 没有非目标。
- 没有失败场景。
- 没有验收标准。
- 把实现细节写死到没有调整空间。

### Execution Plan 的好坏标准

好的 Execution Plan：

- 每个 slice 都能单独验证。
- 每个 slice 的 diff 足够小。
- 写清影响文件。
- 写清先写哪些失败测试。
- 写清局部验证命令和完整 check 命令。
- 写清回滚方式。
- 状态 checkbox 能反映真实进度。

坏的 Execution Plan：

- “实现后端逻辑”这种大块任务。
- “补充必要测试”这种空话。
- 没有验证命令。
- 没有非目标。
- 没有 slice 边界。
- 计划一写完，执行时又重新设计。

### Project Status 的好坏标准

好的 Project Status：

- 新会话读完能继续。
- 当前 active plan 一眼能找到。
- 最近验证结果清楚。
- 阻塞和下一步清楚。
- 过期状态会被更新或删除。

坏的 Project Status：

- 像 changelog。
- 像 roadmap。
- 像任务清单。
- 没有更新时间。
- 写了很多已完成细节，却没有下一步。

## Agent 执行时的协议

当已有 Execution Plan 时，给 agent 的提示词应该强调：

```text
计划负责决定方向。
本轮只执行当前 slice。
不要重新规划大方向。
不要顺手做后续 slice。
如果发现计划错误，先停下来说明冲突。
先写失败测试，再写最小实现。
跑计划里的验证命令。
完成后更新 plan 和 project-status。
```

推荐提示词：

```text
请按照已有计划执行，不要重新规划大方向。

计划文档：
docs/plans/YYYY-MM-DD-feature-name.md

请先读取：
1. AGENTS.md
2. README.md
3. docs/README.md
4. docs/project-status.md
5. docs/development.md
6. docs/testing.md
7. 上面的计划文档
8. 计划点名的相关代码和测试

执行规则：
1. 默认只执行第一个未完成 slice。
2. 如果我指定了 slice，只执行我指定的 slice。
3. 不要做后续 slice。
4. 不要计划外重构。
5. 先写失败测试，并确认因为预期原因失败。
6. 再写最小实现。
7. 跑本 slice 的局部验证命令。
8. 必要时跑完整 check。
9. 更新计划文档状态和 docs/project-status.md。
10. 如果发现计划不成立，停下来说明冲突和建议，不要直接扩大范围。
```

这段提示词的目标不是“管得细”，而是阻止 agent 从 implementer 变回 architect。

## 什么时候不写计划

计划也有成本。不是每次都要写 `docs/plans/*.md`。

可以不写正式计划的情况：

- 改 typo。
- 改一行配置。
- 更新链接。
- 调整文档小段落。
- 已有测试明确覆盖的小修。
- 你愿意直接 review 全部 diff 的低风险改动。

但即使不写计划，也建议保留三个动作：

1. 说清验收标准。
2. 完成前跑最小验证。
3. 总结改动和风险。

如果一个任务满足下面任意条件，就应该写计划：

- 影响多个文件。
- 影响公共接口。
- 涉及数据迁移、权限、安全、计费、删除、网络、部署。
- 需要先讨论方案。
- diff 可能超过人类 10 分钟可 review 的范围。
- agent 容易自行扩大范围。
- 你需要新会话接手。

## 常见反模式

### 反模式一：Roadmap 变成无限 TODO

症状：

```text
- [ ] 支持登录
- [ ] 支持权限
- [ ] 支持导出
- [ ] 支持移动端
- [ ] 支持插件
- [ ] 支持同步
...
```

问题是这里没有优先级、边界、触发条件和暂不做。它看起来像计划，实际只是焦虑列表。

改法：

- 用 `Now / Next / Later`。
- 每个 Now 项写为什么现在做。
- Later 只保留方向，不拆任务。
- 暂时不做单独列出来。

### 反模式二：Execution Plan 太抽象

症状：

```text
1. 实现认证逻辑。
2. 添加错误处理。
3. 补充测试。
4. 更新文档。
```

问题是 agent 仍然需要自己猜：

- 哪些文件？
- 哪些测试？
- 什么叫错误处理？
- 什么叫补充测试？
- 怎么验证？

改法：

- 写明影响文件。
- 写明测试文件。
- 写明先失败的场景。
- 写明验证命令。
- 把大任务拆成 slice。

### 反模式三：Status 和 Plan 混成一坨

症状：

一个 `implementation-plan.md` 里既有半年路线，又有昨天测试输出，又有今天阻塞，又有未来 feature。

问题是新会话读不出什么仍然有效。

改法：

```text
长期方向 -> roadmap.md
当前状态 -> project-status.md
当前任务 -> plans/*.md
长期决策 -> decisions/*.md
```

### 反模式四：计划确认后，实现阶段重新规划

症状：

你先让 agent 写了计划，确认后让它实现。它一进实现阶段又说“我先重新分析一下”，然后改了新方向。

改法：

在实现提示词里写清：

```text
不要重新规划大方向。
如果发现计划有问题，先停下来说明冲突和建议。
```

### 反模式五：测试写在计划里，但执行时不守

症状：

计划里写 TDD，但 agent 先写实现，最后补测试。

改法：

把每个 slice 写成：

```text
- [ ] 写失败测试。
- [ ] 运行测试，确认因为预期原因失败。
- [ ] 写最小实现。
- [ ] 运行测试，确认通过。
```

并在完成报告里要求说明：

```text
失败测试是否先出现？
失败原因是什么？
哪个实现让它通过？
```

## 推荐落地到你的 playbook

你现有的 [`../../agent/playbooks/workflows/new-project-bootstrap.md`](../../agent/playbooks/workflows/new-project-bootstrap.md) 和 [`../../agent/playbooks/templates/project-harness-files.md`](../../agent/playbooks/templates/project-harness-files.md) 已经覆盖 `docs/plans/` 和 `docs/project-status.md`，方向是对的。建议后续迭代时补强三点。

### 1. 给 `docs/plans/README.md` 明确分层

建议写清：

```text
docs/plans/ 保存单个功能、bugfix、迁移或重构的执行计划。
它不是长期 roadmap，也不是 project-status。
长期方向放 docs/roadmap.md。
当前状态放 docs/project-status.md。
长期技术决策放 docs/decisions/。
```

### 2. 新项目初始化时可选创建 `docs/roadmap.md`

不是所有项目一开始都有 roadmap，但长期项目建议有。

最小内容：

```text
项目目标
Now
Next
Later
暂时不做
Active Work
```

### 3. 中大型任务可选创建 `docs/specs/`

对于个人小项目，`docs/plans/*.md` 里带 Spec 章节足够。

但当任务满足下面条件时，建议拆出 `docs/specs/`：

- 需求还需要讨论。
- 影响多个模块。
- 需要长期 review。
- 计划会分多天执行。
- 多个 agent / 多个会话需要引用同一个意图文档。

## 最小可执行建议

如果只保留一套简单规则，可以这样写进项目约定：

```text
1. docs/roadmap.md 记录长期方向，不写执行细节。
2. docs/project-status.md 记录当前状态、新会话接手信息和下一步。
3. docs/plans/*.md 记录单个任务的目标、非目标、reviewable slices、TDD 步骤和验证命令。
4. 复杂任务先写 spec，再写 plan；小任务可以把 spec 合并到 plan。
5. 已确认的 plan 进入执行阶段后，agent 默认只做第一个未完成 slice。
6. 如果实现中发现计划错误，先停下来修计划，不要暗中扩大范围。
7. 每个 slice 完成后，更新 plan 状态和 project-status。
```

一句话版本：

> Roadmap 控制方向，Spec 控制意图，Execution Plan 控制行动，Project Status 控制接手。

## 参考资料

- Anthropic: [Best practices for Claude Code](https://code.claude.com/docs/en/best-practices)
- Jesse Vincent: [How I'm using coding agents in September, 2025](https://blog.fsck.com/2025/10/05/how-im-using-coding-agents-in-september-2025)
- Jesse Vincent: [Superpowers: How I'm using coding agents in October 2025](https://blog.fsck.com/2025/10/09/superpowers)
- obra/superpowers: [writing-plans/SKILL.md](https://github.com/obra/superpowers/blob/main/skills/writing-plans/SKILL.md)
- Harper Reed: [My LLM codegen workflow atm](https://harper.blog/2025/02/16/my-llm-codegen-workflow-atm/)
- Simon Willison: [My LLM codegen workflow atm](https://simonwillison.net/2025/Feb/21/my-llm-codegen-workflow-atm/)
- GitHub Blog: [Spec-driven development with AI](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/)
- Microsoft Developer Blog: [Diving Into Spec-Driven Development With GitHub Spec Kit](https://developer.microsoft.com/blog/spec-driven-development-spec-kit)
- Microsoft Research: [Magentic-One: A Generalist Multi-Agent System for Solving Complex Tasks](https://www.microsoft.com/en-us/research/articles/magentic-one-a-generalist-multi-agent-system-for-solving-complex-tasks/)
- AutoGen Docs: [Magentic-One](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/magentic-one.html)
- Basecamp: [Shape Up](https://basecamp.com/shapeup)
- Simon Willison: [Agentic Engineering Patterns](https://simonwillison.net/guides/agentic-engineering-patterns/)
