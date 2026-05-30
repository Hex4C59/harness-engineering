# Superpowers Skill 设计思想调研

调研日期：2026-05-29

## 这篇文档回答什么

这里的 **Superpowers** 指 Jesse Vincent 维护的开源项目 `obra/superpowers`，不是某一个单独的 prompt，也不是某个模型能力。它是一套给 coding agent 使用的软件开发方法论，打包成一组可组合的 skills / plugins，支持 Claude Code、Codex CLI、Codex App、Gemini CLI、OpenCode、Cursor、GitHub Copilot CLI 等多种 harness。

本文回答：

1. Superpowers 是什么？
2. 它包含哪些 skills？
3. 它背后的设计思想是什么？
4. 它和 harness engineering 有什么关系？
5. 它有什么优点、边界和可借鉴之处？

核心结论：

> Superpowers 的本质是把“资深工程师的工作纪律”编码成 agent 必须遵守的 workflow gates：先设计、再计划、用 TDD 实现、系统性调试、完成前验证、代码审查、隔离分支和 subagent 分工。它不是让模型更聪明，而是让模型更不容易用聪明的方式乱来。

## 来源说明

本次调研主要使用以下来源：

| 来源 | 类型 | 说明 |
|---|---|---|
| `obra/superpowers` | 上游 GitHub 仓库 | Superpowers 的主要源头，README 和 skills 实现最完整 |
| `skills/*/SKILL.md` | skill 源码 | 具体工作流定义，例如 brainstorming、TDD、debugging |
| agentskill.sh 搜索结果 | skill 市场条目 | 用于确认社区搬运版本和命名混淆 |
| Claude / Codex plugin 说明 | 安装入口 | 用于确认它已被打包为多个 agent harness 的插件 |

agentskill.sh 上也能搜到 `qlx288/superpowers`、`mkurman/using-superpowers`、`superpowers-lab` 等条目。其中 `qlx288/superpowers` 是一个中文简介式搬运版本，描述“包含 14 种开发技能”；`mkurman/using-superpowers` 更像单个入口技能；`superpowers-lab` 是配套实验环境。本文重点分析上游 `obra/superpowers`，因为它更完整，也更接近原始设计。

## Superpowers 是什么

Superpowers 是一套 **software development methodology for coding agents**。它不是教 agent 某个框架 API，也不是给模型补充某个领域知识，而是定义 agent 做软件开发时应该如何工作。

它试图解决的问题是：

```text
LLM coding agent 太容易直接写代码、跳过设计、跳过测试、猜测 bug 根因、声称完成但没有验证。
```

Superpowers 的解决方式是把专业开发流程拆成多个 skills，并要求 agent 在对应场景下必须调用这些 skills。

它的默认工作流大致是：

```text
用户提出需求
-> using-superpowers 检查应使用哪些 skills
-> brainstorming 澄清需求并形成设计
-> using-git-worktrees 创建隔离工作区
-> writing-plans 写详细实施计划
-> subagent-driven-development 或 executing-plans 执行计划
-> test-driven-development 约束每个实现步骤
-> requesting-code-review / receiving-code-review 做评审循环
-> verification-before-completion 完成前拿证据
-> finishing-a-development-branch 合并、PR 或清理分支
```

用 harness engineering 的语言说：

```text
Superpowers = workflow harness + skill library + process gates + verification discipline
```

它把原本靠人类工程师自觉遵守的流程，变成 agent 上下文里的显式协议。

## 它包含哪些 skills

上游仓库当前核心 skills 包括 14 个：

| Skill | 作用 |
|---|---|
| `using-superpowers` | 入口技能，要求 agent 在任何任务前先判断是否有适用 skill |
| `brainstorming` | 在实现前澄清需求、探索方案、写设计文档并取得用户确认 |
| `writing-plans` | 把设计拆成细粒度实施计划，要求精确文件、代码、命令和验证步骤 |
| `using-git-worktrees` | 创建隔离开发工作区，避免污染主分支 |
| `test-driven-development` | 强制 Red-Green-Refactor，先写失败测试再写实现 |
| `systematic-debugging` | 遇到 bug 时先做根因分析，不允许猜测式修复 |
| `verification-before-completion` | 完成前必须运行验证命令，用证据支撑完成声明 |
| `requesting-code-review` | 请求代码审查，按严重程度报告问题 |
| `receiving-code-review` | 处理审查反馈，修复真实问题而不是争辩 |
| `subagent-driven-development` | 每个任务派发新 subagent，并做 spec review 和 quality review |
| `dispatching-parallel-agents` | 并行派发 agent 做独立任务 |
| `executing-plans` | 按实施计划分批执行，有检查点 |
| `finishing-a-development-branch` | 完成开发分支，验证、合并、PR 或清理 |
| `writing-skills` | 编写和测试新的 skills |

这些 skills 不是平铺的“功能菜单”，而是一个工作流网络。`using-superpowers` 是入口，`brainstorming` 负责需求到设计，`writing-plans` 负责设计到计划，`TDD/debugging/verification` 负责实现纪律，`review/branch/subagent` 负责工程化协作。

## 入口设计：先找流程，再回答问题

`using-superpowers` 是整个系统的入口。它的思想很强硬：

```text
如果有 1% 可能某个 skill 适用，就必须先调用 skill。
```

它特别反对 agent 的常见自我辩解：

- “这只是一个简单问题。”
- “我先看一下代码再说。”
- “我需要先问澄清问题。”
- “我记得这个 skill 的内容。”
- “用 skill 太重了。”

Superpowers 的判断是：这些想法大多是 agent 逃避流程的开端。它要求在澄清问题、探索代码、执行命令之前，先判断是否应该加载流程 skill。

这个设计的核心不是“让 agent 读更多文档”，而是让 agent 从一开始进入正确工作模式：

```text
用户说 WHAT。
skill 决定 HOW。
```

也就是说，用户说“帮我修 bug”并不意味着 agent 可以直接改代码；它应该触发 systematic-debugging，再进入 TDD 和 verification。

## 设计阶段：反对“直接开写”

`brainstorming` 的定位是：所有 creative work 在实现前都必须经过设计。

它定义了一个 hard gate：

```text
没有展示设计并获得用户批准之前，不允许调用实现技能、不允许写代码、不允许 scaffold 项目。
```

它的流程包括：

1. 探索项目上下文。
2. 如果涉及视觉问题，询问是否使用 visual companion。
3. 一次只问一个澄清问题。
4. 提出 2-3 种方案和取舍。
5. 分段展示设计，并逐段确认。
6. 写设计文档到 `docs/superpowers/specs/...`。
7. 自查占位符、矛盾、歧义和范围。
8. 让用户审查设计文档。
9. 转入 `writing-plans`。

这背后的设计思想是：agent 最容易在需求还没弄清楚时做出“看起来很主动”的错误实现。Superpowers 宁愿前置沟通成本，也要减少后续返工。

它对“简单任务”的态度也很明确：简单任务也需要设计，只是设计可以很短。因为越简单的任务，越容易让未验证假设偷偷进入实现。

## 计划阶段：把任务写到初级工程师也能执行

`writing-plans` 要求把设计拆成 2-5 分钟粒度的任务，并且每个任务都必须包含：

- 精确文件路径。
- 创建/修改/测试哪些文件。
- 实际代码片段。
- 运行什么命令。
- 预期输出是什么。
- 何时提交。

它明确禁止：

- `TODO` / `TBD`。
- “添加适当错误处理”这种空话。
- “写上面的测试”但不给测试代码。
- “类似 Task N”。
- 引用尚未定义的类型、函数或方法。

这是一种很有意思的 agent harness 思想：计划不是给资深工程师看的，而是给“能力强但上下文少、品味差、容易偷懒、不爱测试”的执行 agent 看的。

所以计划不是“方向”，而是“可执行脚本”。

这对 coding agent 很重要，因为 agent 的问题经常不是不会写代码，而是：

- 忘记上下文。
- 跳过边界条件。
- 自行扩大范围。
- 测试写得太笼统。
- 一步做太多导致难以审查。

Superpowers 用超细粒度计划来压缩 agent 的自由度。

## TDD：把纪律写成铁律

`test-driven-development` 是 Superpowers 最强硬的 skill 之一。它的铁律是：

```text
NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST
```

如果 agent 先写了实现再想补测试，Superpowers 的要求是：删除实现，重新从失败测试开始。

它坚持完整 Red-Green-Refactor：

1. RED：写一个最小失败测试。
2. Verify RED：确认它以正确原因失败。
3. GREEN：写最小实现。
4. Verify GREEN：确认测试通过。
5. REFACTOR：只在绿灯后清理。

它反对常见说法：

- “我之后补测试也一样。”
- “这太简单了不用测试。”
- “我手动测过了。”
- “删掉已有实现太浪费。”

从设计思想看，这不是单纯偏爱 TDD，而是用 TDD 解决 agent 的两个核心问题：

1. agent 很会写 plausible code，但不一定对。
2. agent 很会说“应该可以”，但缺少证据。

失败测试是 agent 对需求理解正确的证据；通过测试是实现正确的证据。

## Debugging：禁止猜测式修复

`systematic-debugging` 的铁律是：

```text
NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST
```

它把调试拆成四个阶段：

1. Root Cause Investigation：读错误、稳定复现、检查最近变更、在组件边界加诊断、追踪数据流。
2. Pattern Analysis：找工作样例、读参考实现、列差异、理解依赖。
3. Hypothesis and Testing：提出单一假设，用最小改动验证。
4. Implementation：写失败测试，修根因，验证。

其中最有价值的一点是：如果连续 3 次修复失败，要停止继续打补丁，转而质疑架构或问题建模。

这对 agent 特别重要。LLM agent 很容易进入“猜一个改动、跑一下、不行再猜”的循环，甚至把多个改动打包在一起，最后不知道哪个改变产生了效果。Superpowers 强迫 agent 像做实验一样调试：一次只改一个变量，用证据推进。

## 完成前验证：不允许没有证据的完成声明

`verification-before-completion` 的核心是：

```text
NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE
```

它要求在任何“完成了”“修好了”“测试通过了”“可以提交了”之前，先执行 gate function：

1. Identify：什么命令能证明这个声明？
2. Run：运行完整命令。
3. Read：读完整输出和 exit code。
4. Verify：判断输出是否真的支持声明。
5. Claim：只有这时才可以说完成。

它特别强调：不能相信 agent 自己的成功报告。即使 subagent 说完成了，主 agent 也要检查 diff、运行验证、确认实际状态。

这和 2026 年 coding agent 系统卡里强调的 agentic honesty 完全同向。对人类来说，“没验证就说完成”是粗心；对 agent 来说，这是系统性风险。

## Subagent-driven development：隔离上下文 + 双重审查

`subagent-driven-development` 是 Superpowers 里最像 agent runtime 的部分。

它的模式是：

```text
每个任务一个全新 subagent
-> implementer 实现、测试、提交、自查
-> spec reviewer 检查是否满足计划
-> code quality reviewer 检查实现质量
-> 有问题则回到 implementer 修复
-> 直到通过再进入下一任务
```

它强调：

- subagent 不应该继承主会话历史。
- 主 agent 要把任务所需上下文精确构造好。
- spec review 必须先于 code quality review。
- review 发现问题后必须重新 review。
- 不要并行派发多个会互相冲突的实现 subagents。
- 不要让 subagent 自己去读整个计划文件，而是给它当前任务完整文本。

这个设计很贴近 harness engineering：

- 主 agent 是 orchestrator。
- subagents 是 isolated workers。
- plan 是任务协议。
- reviewer 是质量门。
- git commit 是状态边界。
- test output 是验证证据。

它的代价也很明确：更多 agent 调用、更多 token、更多 review 循环。但收益是更少上下文污染、更强任务聚焦、更早发现偏离。

## Git worktrees：用隔离工作区控制风险

Superpowers 把 `using-git-worktrees` 放在设计批准之后、写计划之前或执行前，说明它把隔离工作区视为默认工程实践。

这背后的思想是：

- agent 不应该直接在主分支上大幅修改。
- 每个功能应该有独立 workspace。
- 可以并行开发多个任务。
- 出问题时更容易丢弃或回滚。
- 更容易做分支级验证和 review。

这也是现代 coding agent 的共同趋势：不是只靠 prompt 控制 agent，而是用 git/worktree/sandbox 这种工程机制限制 blast radius。

## 设计思想总结

### 1. Workflow gates > prompt reminders

Superpowers 不满足于在系统提示里写“请先测试”。它把流程拆成独立 skills，每个 skill 都有触发条件、铁律、检查清单和失败模式。

这比普通 prompt 更强，因为它让 agent 在不同阶段进入不同状态：

```text
需求阶段：brainstorming
计划阶段：writing-plans
实现阶段：TDD
调试阶段：systematic-debugging
完成阶段：verification-before-completion
协作阶段：review / subagent / worktree
```

### 2. 降低 agent 自由度

很多 AI coding 工具追求“agent 自主”。Superpowers 的方向更像“受纪律约束的自主”。

它允许 agent 执行复杂任务，但不允许：

- 没有设计就写代码。
- 没有失败测试就写实现。
- 没有根因就修 bug。
- 没有证据就说完成。
- 没有 review 就继续推进。

它的判断是：agent 的问题不是能力不足，而是太容易走捷径。

### 3. 把隐性工程经验显性化

资深工程师做事会自然地：

- 先确认需求。
- 缩小范围。
- 找现有模式。
- 写测试。
- 读完整错误。
- 做最小修复。
- 跑验证。
- 请别人 review。

Superpowers 把这些隐性习惯写成机器可执行的文档协议。

### 4. 用证据替代信任

Superpowers 一直在压制 agent 的“自信表达”：

- 说测试通过之前必须跑测试。
- 说修好了之前必须验证原始症状。
- 说 subagent 完成之前必须检查实际 diff。
- 说符合需求之前必须逐项对照 spec。

这和 harness engineering 的核心一致：

```text
不要相信 agent 的叙述，要看 trace、diff、命令输出和测试结果。
```

### 5. 把复杂任务拆给隔离 agent

Superpowers 不把 subagent 当成“越多越好”的并行魔法，而是把它们当成有边界的执行 worker。

每个 worker 得到明确任务和上下文，完成后由 reviewer 检查。这比在一个长上下文里让同一个 agent 做所有事更稳。

## 和 harness engineering 的关系

Superpowers 是一个非常具体的 harness engineering 案例。

| Harness engineering 概念 | Superpowers 对应设计 |
|---|---|
| 上下文治理 | 每个 skill 按阶段加载；subagent 获得精确上下文 |
| 工具边界 | worktree、git、测试命令、review 流程 |
| 状态管理 | design doc、plan doc、checkbox、commit、branch |
| 验证反馈 | TDD、测试命令、完成前验证 |
| 人类介入 | 设计审批、spec review、执行策略选择 |
| 安全边界 | worktree 隔离、禁止主分支直接实现、完成前检查 |
| 可观测性 | 要求保留命令输出、diff、review 结果 |
| 多 agent 编排 | subagent-driven-development、parallel agents |

它证明了一个重要观点：harness 不一定必须是大型服务端 runtime。一个精心设计的 skill suite，也可以成为轻量级 harness，把 agent 行为约束到专业工程流程里。

## 优点

### 1. 对抗 agent 的坏习惯

Superpowers 专门针对 LLM coding agent 的常见坏习惯：

- 直接写代码。
- 猜测用户意图。
- 跳过测试。
- 猜 bug 根因。
- 过度实现。
- 没验证就宣布成功。
- 在主分支乱改。
- 把 review 当形式。

这些都是实际使用 coding agent 时最常见、也最浪费时间的问题。

### 2. 非常适合复杂任务

对简单问题来说，它可能显得很重。但对大型功能、复杂 bug、长任务 refactor、多人协作式 agent 工作流，它能显著降低失控概率。

它尤其适合：

- 新功能开发。
- 多文件重构。
- 难复现 bug。
- 需要长期验证的任务。
- 希望 agent 自主工作数小时但仍不偏航的场景。

### 3. 可跨 harness 移植

Superpowers 支持多种 agent 平台，说明它的核心不是某个具体工具 API，而是流程模式。

当然，不同 harness 的工具名不同，需要适配：

- Claude Code 的 Skill tool。
- Gemini CLI 的 activate_skill。
- Codex / Copilot / OpenCode 的插件或工具映射。

但底层思想可以迁移到任何 coding agent。

## 边界和风险

### 1. 对小任务可能过重

“任何 creative work 都必须先 brainstorming、写 design doc、用户审批”对大型功能很合理，但对改错别字、小配置、小文档修订可能太重。

上游 skill 强调简单任务也要设计，只是可以很短。这体现了它的强纪律立场，但在日常使用中需要根据用户偏好调整。

### 2. TDD 铁律不适合所有改动

TDD 对业务逻辑、bugfix、行为变化很有价值。但对探索性原型、UI 微调、配置迁移、生成代码、文档编辑，不一定总是最优。

上游 skill 也允许例外，但要求询问 human partner。

### 3. 成本和延迟更高

Subagent-driven development 每个任务至少可能涉及：

- implementer。
- spec reviewer。
- code quality reviewer。
- 多轮修复。

这会带来更多 token、更多模型调用、更长执行时间。它适合高价值任务，不适合所有任务。

### 4. Skill 本身也可能过时

`using-superpowers` 强调“不要凭记忆，读取当前 skill”。这是对的。skill 是活文档，平台工具、模型能力、最佳实践都会变。

如果把 skill 当成永久真理，就会变成另一种僵化流程。

### 5. 强硬流程可能和用户指令冲突

Superpowers 明确规定用户指令优先级最高。如果用户的 `AGENTS.md` 或直接请求说“不需要 TDD”或“只做快速草稿”，应该听用户的。

这点很重要：workflow 是服务用户目标的，不是反过来。

## 对 playbooks 工具箱的启发

这篇文档现在最直接的落点不是整个仓库，而是 [`docs/coding-agents/agent/`](../../agent/README.md) 这套轻量 **Coding Agent Harness Toolkit**，尤其是其中的 [`playbooks/`](../../agent/playbooks/README.md) 和 [`skills/`](../../agent/skills/README.md)。

Superpowers 是重型 skill suite：它希望 agent 在运行时自动或半自动选择 skills，并强制进入对应 workflow。当前 playbooks 更轻量：默认由人先选入口，再让 agent 按需读取 workflow、prompt、checklist 或 principle；只有边界清楚、复用价值高的流程才升级成 skill。

这两者的关系不是“照搬 Superpowers”，而是：

```text
Superpowers 提供设计校准。
Playbooks 提供更轻量、更可复制、更适合个人项目的落地形态。
Skills 承接已经成熟、可触发、可复用的 gate。
```

## Superpowers 到 playbooks / skills 的映射

| Superpowers 设计 | 本工具箱对应位置 | 取舍 |
|---|---|---|
| `using-superpowers` 入口 skill | `playbooks/README.md` 的场景入口表 | Superpowers 让 agent 先选 skill；playbooks 先让人选 workflow，再让 agent 按需读 |
| `brainstorming` | `prompts/solution-comparison.md`、`prompts/tech-stack-selection.md`、`principles/dependency-and-architecture-changes.md` | 不要求所有小任务都 brainstorming；只在需求、架构或方案不清时触发 |
| `writing-plans` | `principles/task-document-layers.md`、`prompts/reviewable-slices.md`、`prompts/execute-plan-slice.md` | 保留 Spec / Plan / Status 分层，但按任务大小决定轻重 |
| `test-driven-development` | `principles/tdd-and-verification.md`、`prompts/new-feature-tdd.md`、`checklists/daily-development.md` | 把 TDD 作为默认纪律，但允许文档、小配置、探索任务用替代验证 |
| `systematic-debugging` | `workflows/bugfix.md`、`prompts/bugfix.md` | bugfix 必须先复现、找根因、写回归测试或说明不可自动化 |
| `verification-before-completion` | `workflows/review-and-finish.md`、`checklists/review-and-finish.md` | 完成声明前必须有 fresh validation evidence |
| `requesting-code-review` / `receiving-code-review` | `skills/review-gate/`、`prompts/review-gate.md`、`prompts/independent-review.md` | review gate 已升级成 skill；只读审查 correctness、验收、测试、兼容性、安全和越界问题 |
| commit boundary discipline | `skills/commit-gate/` | commit gate 独立于 review gate：review gate 管质量，commit gate 管历史边界 |
| `using-git-worktrees` | 未来可进入 `workflows/` 或 `principles/` | 当前 playbooks 还没有系统化 worktree 策略，这是可补强点 |
| `subagent-driven-development` | `skills/review-gate/references/reviewer-agent.md` 已吸收 reviewer subagent 模式；完整实现编排暂不默认 | 当前先做只读 reviewer subagent，不默认多 agent 并行实现 |
| `writing-skills` | `skills/` 和 `meta/extension-guide.md` | 成熟、重复、高价值流程再升级为 skill，不一开始就 skill 化 |

## 对 playbooks / skills 的设计建议

### 1. 不把所有流程都升级成 skill

Superpowers 把流程做成 skills，这是它的强项。但当前工具箱更适合保留三层：

```text
先写成 prompt / workflow
-> 用真实任务验证
-> 重复、高价值、边界清楚后再升级为 skill
```

例如 [`commit-gate`](../../agent/skills/commit-gate/SKILL.md) 和 [`review-gate`](../../agent/skills/review-gate/SKILL.md) 适合 skill 化，因为它们有明确触发场景、输入、输出、禁止动作和判断标准。相比之下，`learning-first`、`dependency-and-architecture-changes` 更像原则和决策框架，暂时不必变成 skill。

### 2. 把 gates 落到 checklists 和 skills，而不是只写在原则里

Superpowers 最值得借鉴的是 gate 思维：

```text
没有设计，不进入实现。
没有失败测试，不写生产代码。
没有验证证据，不声明完成。
没有 review，不继续扩大范围。
```

在本工具箱里，gate 分两层：

- `checklists/`：日常任务中的阶段放行条件，例如 `daily-development.md`、`review-and-finish.md`、`maintainability-gate.md`。
- `skills/`：高复用、可触发、需要稳定输出格式的 gate，例如 `review-gate` 和 `commit-gate`。

原则文档解释为什么，checklist 决定能不能放行，skill 负责可复用执行。

### 3. 保留轻量路径，避免流程压过任务价值

Superpowers 的流程很强硬，对复杂任务很有价值，但对小修、文档修订、简单配置可能太重。

playbooks 应该明确保留分层：

| 任务类型 | 推荐路径 |
|---|---|
| 小修 / typo / 文档轻改 | 读相关文件 -> 最小修改 -> 局部验证 -> 完成前检查 |
| bugfix | 复现 -> 根因 -> 回归测试或替代验证 -> 最小修复 -> 验证 |
| 中等新功能 | 短 plan -> TDD -> reviewable slice -> check |
| 架构 / 依赖 / 跨模块变化 | Spec / decision -> 用户确认 -> Execution Plan -> 分 slice 实现 |
| 探索 / 陌生技术栈 | scout / learning-first，不直接落地大 diff |

这比 Superpowers 更适合个人长期项目：既保留纪律，又不过度仪式化。

### 4. 把“人选入口，agent 按需读”写成默认模式

Superpowers 入口 skill 要求 agent 主动判断该用哪个 skill。playbooks 当前更稳的模式是：

```text
人先判断当前任务类型
-> 选择 workflow
-> workflow 链接 prompt / checklist / principle
-> 必要时触发 skill
-> agent 只读取当前任务需要的最小材料
```

这能避免 agent 一次性读完整工具箱，也符合 `playbooks/README.md` 里“谁来读”的分层。

### 5. 下一步最值得补的是 worktree / parallel agent 边界

Superpowers 在 worktree 和 subagent 编排上比当前 playbooks 更完整。本工具箱已经先吸收了 reviewer subagent 模式：`review-gate` 会优先派独立 reviewer agent，并把 reviewer 规则拆到 `references/reviewer-agent.md`。

后续更值得补的是两个轻量边界文件：

- `principles/worktree-and-branch-isolation.md`：什么时候需要 worktree、什么时候普通分支足够、什么时候不要并行。
- `workflows/parallel-research-and-review.md`：research / POC / review 可以并行，核心实现默认不并行。

不建议一开始复制 Superpowers 的完整 subagent-driven-development。先把边界写清楚，再用真实任务验证。

## 一个适合 playbooks 的简化版 Superpowers 流程

如果不安装完整 Superpowers，也可以把它吸收成 playbooks 默认循环：

```text
1. Clarify
   先确认目标、约束、成功标准。

2. Design
   给出 1-3 个方案和推荐方案，用户确认。

3. Plan
   拆成小任务，每个任务有文件、命令、验证方式。

4. Isolate
   用分支、worktree、临时目录或 sandbox 隔离修改。

5. Test First
   行为变化先写失败测试。

6. Implement
   最小改动，避免顺手重构。

7. Verify
   跑命令，看输出，用证据说明状态。

8. Review
   对照需求和代码质量做检查。

9. Finish
   汇总 diff、验证结果、风险和下一步。
```

这就是 Superpowers 对 playbooks 的核心启发：把 agent 从“聪明的自动补全”变成“受 workflow、gate、验证和 review 约束的协作者”。

## 参考资料

- GitHub: [obra/superpowers](https://github.com/obra/superpowers)
- Superpowers README: [Superpowers overview](https://github.com/obra/superpowers/blob/main/README.md)
- Release announcement: [Superpowers](https://blog.fsck.com/2025/10/09/superpowers/)
- Skill source: [using-superpowers](https://github.com/obra/superpowers/blob/main/skills/using-superpowers/SKILL.md)
- Skill source: [brainstorming](https://github.com/obra/superpowers/blob/main/skills/brainstorming/SKILL.md)
- Skill source: [writing-plans](https://github.com/obra/superpowers/blob/main/skills/writing-plans/SKILL.md)
- Skill source: [test-driven-development](https://github.com/obra/superpowers/blob/main/skills/test-driven-development/SKILL.md)
- Skill source: [systematic-debugging](https://github.com/obra/superpowers/blob/main/skills/systematic-debugging/SKILL.md)
- Skill source: [verification-before-completion](https://github.com/obra/superpowers/blob/main/skills/verification-before-completion/SKILL.md)
- Skill source: [subagent-driven-development](https://github.com/obra/superpowers/blob/main/skills/subagent-driven-development/SKILL.md)
- agentskill.sh: [qlx288/superpowers](https://agentskill.sh/qlx288/superpowers)
- agentskill.sh: [mkurman/using-superpowers](https://agentskill.sh/mkurman/using-superpowers)

## 最短总结

Superpowers 的设计思想可以压缩成一句话：

> 不要让 coding agent 凭感觉写代码；给它一套必须经过的工程流程，让设计、计划、测试、调试、验证、review 和分支管理成为显式状态机。

这正是 harness engineering 的一个小而锋利的样本：模型能力负责生成和推理，skill/harness 负责约束流程、保存状态、降低风险、要求证据。
