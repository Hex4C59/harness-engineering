# 什么时候该用 Subagent

调研日期：2026-05-30

## 这篇文档回答什么

现在很多 coding agent 都支持 `subagent`、agent teams、fork、handoff、agent-as-tool、parallel agents。你一直没用它，其实很正常：subagent 不是“高级用户必须打开的开关”，而是一种 **上下文隔离和任务分工工具**。

本文回答：

1. 什么是 subagent，和普通新窗口、并行 agent、handoff 有什么区别。
2. 为什么要用 subagent。
3. 哪些场景值得用，哪些场景不值得用。
4. 个人 coding / 写作 / 调研工作流里怎么开始用。
5. 怎么避免 subagent 带来的成本、冲突和 review 债。

核心结论：

> Subagent 的主要价值不是“让更多 AI 一起干活”，而是把高噪声、高 token、高专门性的子任务隔离出去，让主 agent 保持清醒、保持任务所有权。它最适合用于代码库调研、长日志/测试输出分析、独立方案比较、专项 review、安全审查、并行 research 和低耦合机械任务；不适合小改动、强耦合实现、需求不清、没有验证手段或你没有时间 review 的任务。

一句话：

```text
主 agent 管目标、计划、合成和最终判断。
subagent 管局部探索、专项分析和可隔离执行。
人类管边界、取舍、review 和是否合并。
```

## 先给操作答案

### 什么时候值得用

| 场景 | 是否用 subagent | 原因 |
|---|---:|---|
| 大代码库里找相关实现 | 推荐 | 搜索会读很多文件，污染主上下文 |
| 跑长测试、看长日志、分析 CI 输出 | 推荐 | verbose 输出留在子上下文，只返回失败摘要 |
| 让多个 agent 比较方案 | 推荐 | 独立视角能减少路径依赖 |
| 安全、性能、可访问性、测试覆盖专项 review | 推荐 | 专家 prompt + 限定工具更聚焦 |
| 调研某个库/论文/产品能力 | 推荐 | research 天然可并行 |
| 同一功能下多个互不依赖文件改动 | 谨慎推荐 | 可以提速，但要控制冲突和 review |
| 小 typo、小文案、单文件小改 | 不推荐 | 调度成本高于收益 |
| 需求还不清楚 | 不推荐 | 会把不清楚的问题复制给多个 agent |
| 多个 subagent 改同一核心模块 | 不推荐 | 冲突和 review 债会放大 |
| 你没有时间审结果 | 不推荐 | 并行越多，review 债越大 |

### 最小使用规则

```text
能直接做的小任务，直接做。
会污染主上下文的探索，派 subagent。
会产生大量输出的命令，派 subagent。
需要不同视角的评审，派 subagent。
会改代码的 subagent，必须有边界、验证和 review。
```

## 什么是 subagent

在 coding agent 语境里，subagent 通常指：

```text
由主 agent / 用户派生出来的一个独立 agent 实例，
它有自己的上下文窗口、指令、工具权限和模型配置，
完成一个子任务后，把摘要或结果返回给主 agent。
```

不同工具叫法不同：

| 名称 | 大致含义 |
|---|---|
| subagent | 独立上下文的子 worker |
| fork | 继承当前会话历史，但工具调用留在分支上下文 |
| handoff | 把对话控制权交给另一个 specialist |
| agent-as-tool | 主 agent 仍掌控最终回答，把 specialist 当工具调用 |
| parallel agent | 多个 agent 同时执行，可能是 subagent，也可能是多个窗口/worktree |
| agent team | 多个长期 worker，各自有独立上下文和任务状态 |

在个人 coding 工作流里，你不用先纠结名字。先记这个模型：

```text
subagent = 带独立上下文的临时专家
```

## 为什么要用 subagent

### 1. 隔离上下文污染

Anthropic 的 Claude Code 文档和 best practices 都反复强调：context window 是 coding agent 的核心约束。代码库调研、日志分析、测试输出、搜索结果会消耗大量上下文。

Subagent 的好处是：

```text
脏活累活在子上下文里做。
主窗口只拿结论、文件引用和下一步建议。
```

这和你刚整理的窗口管理文档完全同向：不要把所有信息都摊在主工作台上。

### 2. 并行探索，减少等待

Anthropic 的 multi-agent research system 把 subagent 当作“智能过滤器”：不同 subagent 同时探索不同方向，然后把最重要的信息压缩给 lead agent。

对 coding 来说，类似场景是：

```text
一个 subagent 查认证模块。
一个 subagent 查数据库模型。
一个 subagent 查测试约定。
主 agent 最后综合。
```

这比主 agent 顺序读所有东西更快，也能减少单一路径带来的偏见。

### 3. 专项角色更聚焦

Subagent 可以有专门 prompt、工具和模型，比如：

- `security-reviewer`：只看安全问题。
- `test-runner`：只跑测试和总结失败。
- `performance-reviewer`：只看性能热点。
- `docs-researcher`：只调研资料并给来源。
- `implementation-planner`：只写计划，不改代码。

角色分工本身不是魔法，但它能减少主 agent 在同一个窗口里反复切换心智模式。

### 4. 限制工具权限

Claude Code 的 subagents 和 OpenAI Agents SDK 都支持给不同 agent 不同工具/权限。比如：

```text
reviewer: Read / Grep / Glob，只读
test-runner: Bash / Read / Grep，可以跑测试
implementer: Edit / Bash / Read，可以改代码
security-reviewer: Read / Grep / Glob，不允许联网和写文件
```

这比只在 prompt 里说“不要改文件”更可靠。

### 5. 提供独立视角

Jesse Vincent 的 architect / implementer 流程很典型：一个会话写计划，另一个会话实现，再回到 architect review。Superpowers 后来把这个流程产品化：subagent 实现后，还要经过 spec review 和 code quality review。

这背后的价值是：

```text
写代码的上下文和审代码的上下文分离。
审查者不被实现过程里的借口和失败路线污染。
```

## 论文和技术报告怎么说

### AutoGen：多 agent 是可编程 conversation pattern

Microsoft AutoGen 论文把多个 agent 视为可以互相对话、使用工具、接入人类反馈的可组合单元。它的启发不是“让 agent 互聊越多越好”，而是：

```text
多 agent 需要明确的 interaction behavior。
自然语言和代码都可以用来定义协作模式。
```

对个人使用的启发：不要只说“派几个 agent 看看”。要明确每个 agent 的输入、输出和停止条件。

### Magentic-One：必须有 orchestrator、计划和进度账本

Magentic-One 是 Microsoft 的通用多 agent 系统。它有一个 Orchestrator，负责：

- 制定计划。
- 维护 task ledger。
- 给 specialist 分配任务。
- 跟踪 progress ledger。
- 发现卡住时重新规划。
- 判断什么时候完成。

这对 coding agent 很重要：

```text
subagent 不等于自治团队。
没有 orchestrator，多 agent 只会制造更多状态。
```

在你的个人工作流里，orchestrator 可以是：

- 你本人。
- 主 agent。
- 一份 `docs/plans/*.md`。
- 或者三者组合。

### MetaGPT：角色分工要配 SOP 和结构化交接

MetaGPT 把软件开发拆成产品经理、架构师、工程师等角色，并强调 SOP、结构化消息和中间产物。它的核心启发是：

```text
角色名不重要。
角色之间交接什么 artifact 才重要。
```

如果 subagent 只返回“我看了，没问题”，价值很低。更好的返回是：

```text
发现列表
文件路径
证据
风险等级
建议修改
验证命令
不确定项
```

### ChatDev：多 agent 会带来沟通问题

ChatDev 展示了多 agent 用自然语言协作做软件开发的潜力，也暴露了风险：角色翻转、重复指令、假回复、无效沟通、功能扩张。

对个人使用的警告：

```text
subagent 输出必须短、结构化、可验证。
不要让多个 agent 开放式闲聊。
不要让它们自己决定扩大需求。
```

### Anthropic Research：subagent 是上下文工程

Anthropic 的 context engineering 文章把 sub-agent architecture 放在 compaction、structured note-taking 旁边。它不是单纯并行加速，而是一种绕开上下文限制的系统设计：

```text
lead agent 保留高层计划。
subagent 消耗大量 token 做局部工作。
返回 1k-2k tokens 的浓缩结果。
```

这正是个人 coding agent 最容易受益的地方。

## 和其他机制的区别

### Subagent vs 新窗口

| 机制 | 适合 |
|---|---|
| 新窗口 | 新任务、新主题、独立 review、长期切换 |
| subagent | 当前任务里的局部探索或专项检查 |

新窗口通常由你人工接管；subagent 通常由主 agent 调用并把结果带回来。

### Subagent vs compact

| 机制 | 适合 |
|---|---|
| compact | 当前长会话继续往下做 |
| subagent | 把某个高噪声子任务隔离出去 |

Compact 是整理旧上下文。Subagent 是避免脏上下文进入主窗口。

### Subagent vs handoff

OpenAI Agents SDK 的区分很清楚：

| 模式 | 谁拥有最终回答 |
|---|---|
| agents as tools | 主 agent 拥有最终回答，调用 specialist 获取帮助 |
| handoff | specialist 接管这段对话 |

Coding 工作流里，多数时候你需要的是 `agent-as-tool` 式 subagent：主 agent 仍然负责最终计划和改动。

### Subagent vs 并行 agent

Subagent 强调独立上下文和被主 agent 调度。并行 agent 可以是：

- 多个 subagent。
- 多个终端窗口。
- 多个 worktree。
- 多个云端 background agents。

如果会改代码，最好用 worktree / branch / sandbox 隔离。

## 适合 subagent 的典型模式

### 模式一：Scout，先探路

适合：

- 大代码库。
- 你不知道该改哪里。
- 有多个可能入口。

Prompt：

```text
请派一个 subagent 做 scout，不要改代码。

任务：

请它只做：
1. 找相关文件和入口。
2. 总结当前实现路径。
3. 标出可能风险和缺口。
4. 给出建议修改点。

返回格式：
- 相关文件
- 关键发现
- 风险
- 建议下一步
- 不确定项
```

### 模式二：长日志 / 测试输出分析

适合：

- 测试输出很长。
- CI log 很长。
- 错误堆栈很多。

Prompt：

```text
请用 subagent 跑测试并分析输出。

要求：
1. 可以运行测试命令。
2. 不要修改代码。
3. 不要把完整日志带回主窗口。
4. 只返回失败测试、错误摘要、最可能根因、相关文件、建议复现命令。
```

### 模式三：专项 review

适合：

- 改动已经完成。
- 需要第二视角。
- 需要安全、性能、可访问性、测试覆盖等专项关注。

Prompt：

```text
请派一个只读 subagent review 当前 diff。

关注：
- 正确性
- 安全风险
- 测试缺口
- 兼容性

不要报告纯风格偏好。
每个发现必须包含：严重程度、文件路径、证据、建议修复。
```

### 模式四：并行方案比较

适合：

- 架构选择。
- 技术选型。
- 不确定怎么切分实现。

Prompt：

```text
请派 3 个 subagent 独立评估这个方案，不要互相读取对方结论：
1. 简单实现派。
2. 可维护性派。
3. 风险/测试派。

每个 subagent 输出：
- 推荐方案
- 反对理由
- 影响文件
- 测试方式
- 最大风险

最后由主 agent 综合，不要投票式平均，要给出取舍判断。
```

### 模式五：低耦合并行实现

适合：

- 明确计划已经写好。
- 子任务互不改同一文件。
- 有测试和 review。
- 最好有 worktree / branch 隔离。

Prompt：

```text
请判断这些任务是否可以并行派给 subagents。

只有满足以下条件才允许并行：
1. 不改同一文件。
2. 不改共享公共接口。
3. 每个任务有明确验收标准。
4. 每个任务有独立验证命令。
5. 主 agent 最后必须 review diff 并跑总验证。

如果不满足，请顺序执行。
```

### 模式六：写作和调研分工

这个项目里很适用。

可以这样分：

```text
research subagent：查资料，只返回来源和要点。
structure subagent：提出文档大纲。
critic subagent：审查逻辑、重复、缺口。
main agent：写最终文档并更新 README。
```

注意：最终引用和结论仍要由主 agent 统一整理，不要把多个 subagent 摘要直接拼接。

## 不适合用 subagent 的情况

### 1. 小任务

例如：

- 改一个链接。
- 改标题。
- 补一行 README。
- 单文件小修。

这类任务直接做更快。

### 2. 强耦合实现

例如：

- 改核心状态机。
- 改公共类型和多个调用点。
- 数据库迁移 + API + UI 同时改。

这类任务可以用 subagent 调研和 review，但不建议多个 subagent 同时实现。

### 3. 需求不清楚

如果你自己还没说清目标，派 subagent 只会让不确定性扩散。

正确顺序：

```text
先和主 agent clarify。
再写计划。
再决定是否派 subagent。
```

### 4. 没有验证手段

如果没有测试、lint、可运行命令、截图或人工验收标准，subagent 只会更快地产出“看起来对”的东西。

### 5. 你没有 review 时间

Simon Willison 和 Denny Britz 都提醒过：并行 agent 的瓶颈不是生成，而是 review。你只能合并你能理解和验证的东西。

## 推荐的个人使用层级

如果你之前从没用过 subagent，不要一上来搞 agent team。按这四级来：

### Level 1：只读 scout

先让 subagent 做代码库调研，不改文件。

收益最大，风险最小。

### Level 2：只读 reviewer

让 subagent review diff、找测试缺口、安全风险。

你保留最终判断。

### Level 3：test/log runner

让 subagent 跑长测试、读日志，只返回摘要。

注意给它明确命令和输出格式。

### Level 4：隔离实现 worker

只有在任务明确、文件不冲突、有 worktree 或分支、有验证命令时，才让 subagent 改代码。

## 给 `AGENTS.md` 的可迁移规则

如果要把 subagent 使用规范写进项目，可以用这段：

```markdown
## Subagent 使用边界

允许使用 subagent：

- 代码库调研、文件定位、方案比较。
- 长测试、长日志、CI 输出分析。
- 只读 code review、安全 review、性能 review、测试覆盖 review。
- 互不修改同一文件、验收标准明确、验证命令明确的低耦合子任务。

默认不要使用 subagent：

- 小改动。
- 需求不清的任务。
- 多个 subagent 同时改同一文件或共享接口。
- 没有测试或验证方式的实现任务。
- 人类没有时间 review 的并行实现。

规则：

- 主 agent 必须保留最终任务所有权。
- subagent 返回必须结构化，包含文件路径、证据、不确定项和建议下一步。
- 只读 review subagent 不允许改文件。
- 实现 subagent 的改动必须由主 agent 检查 diff 并运行验证命令。
- 不要相信 subagent 的“完成”声明，必须用测试、lint、diff 或人工验收验证。
```

## 一个完整示例

场景：你要给既有项目加一个认证相关功能，但不熟悉现有代码。

比较稳的流程：

```text
1. 主 agent 读取 AGENTS.md / README / docs。
2. 主 agent 写目标、非目标、验收标准。
3. subagent A 调研 auth/token/session 现有实现。
4. subagent B 调研数据库/user model/migration 约定。
5. subagent C 调研测试风格和现有 auth tests。
6. 主 agent 综合三个摘要，写计划。
7. 人类确认计划。
8. 主 agent 或单个 implementer 执行第一个 slice。
9. test-runner subagent 跑相关测试并总结失败。
10. reviewer subagent 做只读 diff review。
11. 主 agent 修复、跑总验证、总结。
```

这比让一个 agent 从头到尾读所有文件更可控。

## Subagent 输出格式模板

为了避免 subagent 回来一大段散文，可以强制格式：

```markdown
## 结论

## 证据

- 文件：
- 行为：
- 命令：

## 关键发现

1.
2.
3.

## 风险

## 建议下一步

## 不确定项
```

对 review subagent：

```markdown
## Findings

| Severity | File | Issue | Evidence | Suggested fix |
|---|---|---|---|---|

## Test gaps

## Residual risk
```

对 research subagent：

```markdown
## Sources

| Source | Type | Date | Relevance |
|---|---|---|---|

## Facts

## Inferences

## Limitations

## Recommended framing
```

## 常见误区

### 误区 1：subagent 越多越强

不是。Subagent 会增加 token、协调成本、状态复杂度和 review 债。

### 误区 2：subagent 可以替代主 agent 判断

不能。Subagent 是 worker，不是产品负责人，也不是最终 reviewer。

### 误区 3：每个角色都要一个常驻 subagent

不需要。先从临时 scout / reviewer 开始。只有重复出现的任务，才值得做成 custom subagent。

### 误区 4：多个 subagent 聊一聊会自动变聪明

很多论文和系统都说明，多 agent 需要 SOP、结构化交接、orchestrator 和停止条件。随便聊天容易产生重复、跑偏和幻觉级联。

### 误区 5：subagent 说完成就完成

不算。必须看 diff、跑测试、读输出。

## 一句话模型

可以这样记：

```text
Subagent 不是更多脑子。
Subagent 是更干净的工作台、更窄的工具箱、更明确的角色。
```

什么时候用：

```text
当子任务会消耗大量上下文、需要独立视角、可以并行、或适合更窄权限时，用 subagent。
当任务很小、强耦合、需求不清、无法验证、或你没时间 review 时，不用。
```

## 参考资料

- Anthropic Docs: [Create custom subagents](https://docs.anthropic.com/en/docs/claude-code/sub-agents)
- Anthropic: [Best practices for Claude Code](https://www.anthropic.com/engineering/claude-code-best-practices)
- Anthropic: [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- Anthropic: [How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)
- OpenAI Agents SDK: [Agent orchestration](https://openai.github.io/openai-agents-python/multi_agent/)
- OpenAI API: [Orchestration and handoffs](https://developers.openai.com/api/docs/guides/agents/orchestration)
- OpenAI Agents SDK: [Handoffs](https://openai.github.io/openai-agents-python/handoffs/)
- OpenAI Cookbook: [Orchestrating Agents: Routines and Handoffs](https://developers.openai.com/cookbook/examples/orchestrating_agents)
- Microsoft / AutoGen: [AutoGen: Enabling Next-Gen LLM Applications via Multi-Agent Conversation](https://arxiv.org/abs/2308.08155)
- Microsoft: [Magentic-One: A Generalist Multi-Agent System for Solving Complex Tasks](https://arxiv.org/abs/2411.04468)
- Sirui Hong et al.: [MetaGPT: Meta Programming for A Multi-Agent Collaborative Framework](https://arxiv.org/abs/2308.00352)
- Chen Qian et al.: [ChatDev: Communicative Agents for Software Development](https://arxiv.org/abs/2307.07924)
- Simon Willison: [Subagents - Agentic Engineering Patterns](https://simonwillison.net/guides/agentic-engineering-patterns/subagents)
- Simon Willison: [Embracing the parallel coding agent lifestyle](https://simonwillison.net/2025/Oct/5/parallel-coding-agents/)
- Denny Britz: [Thoughts on coding agents](https://dennybritz.com/posts/coding-agents)
- Jesse Vincent: [How I'm using coding agents in September, 2025](https://blog.fsck.com/2025/10/05/how-im-using-coding-agents-in-september-2025)
- Jesse Vincent: [Superpowers: How I'm using coding agents in October 2025](https://blog.fsck.com/2025/10/09/superpowers)
