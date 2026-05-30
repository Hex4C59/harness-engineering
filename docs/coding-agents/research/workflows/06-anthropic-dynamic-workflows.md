# Anthropic Dynamic Workflows 调研

调研日期：2026-05-31

这篇文档整理 Anthropic / Claude Code 新近提出的 **Dynamic workflows**。这里的重点不是泛泛地说“AI workflow 是动态的”，而是 Claude Code 官方文档中一个具体功能：由 Claude 为当前任务写出可执行 workflow script，再由 runtime 在后台编排大量 subagents。

本文区分：

- `事实`：官方文档或明确来源中直接描述的内容。
- `推断`：基于多个官方说明归纳出的工程含义。
- `个人整理框架`：面向本项目 `Coding Agent Harness Toolkit` 的落地建议。

## 核心结论

`事实` Claude Code 的 Dynamic workflows 是一个 research preview 功能。官方把它描述为：Claude 为用户描述的任务写一个 JavaScript orchestration script，workflow runtime 在后台执行这个 script，并用它编排大量 subagents。适用场景包括 codebase-wide audit、大规模迁移、需要交叉核验的 research，以及需要从多个角度起草和比较的复杂计划。

`推断` 它的核心创新不是“多开几个 subagent”，而是把编排状态从主对话上下文移到可执行脚本里：

```text
普通 subagents / skills：
主对话里的 Claude 按 turn 决定下一步，结果回到 Claude context。

Dynamic workflows：
Claude 先生成 workflow script；
script 持有循环、分支、中间结果和阶段安排；
runtime 执行 script；
主对话主要接收最终结果。
```

这意味着 Dynamic workflows 是一种更强的 agent harness：

```text
LLM 负责生成任务专属编排；
workflow script 负责稳定执行编排；
subagents 负责并行探索、修改或核验；
runtime 负责后台运行、进度、暂停、恢复和管理。
```

## 术语边界

`事实` Anthropic 在《Building effective agents》中已经区分过 workflow 和 agent：workflow 是沿预定义路径编排 LLM 和工具；agent 是 LLM 动态决定流程和工具使用。

`事实` Claude Code Dynamic workflows 进一步把两者合在一起：workflow 的路径不是人手写死的，而是 Claude 根据任务动态写出脚本；但脚本一旦生成，执行阶段由 runtime 按脚本编排。

`推断` 所以这里的 dynamic 不是“完全自由发挥”，而是：

```text
生成阶段是动态的：Claude 根据任务写 workflow。
执行阶段是结构化的：runtime 按 script 调度 subagents。
复用阶段是稳定的：满意的 workflow 可以保存成命令反复运行。
```

这和你之前讨论的 orchestration map 不冲突。你的 orchestration map 是“何时使用什么 gate”的路线选择；Dynamic workflows 是“当任务大到需要大量 agent 时，如何执行这个编排”的 runtime 机制。

## 官方功能摘要

### 1. 可用性

`事实` 截至 2026-05-31，Claude Code 文档称 Dynamic workflows：

- 处于 research preview。
- 需要 Claude Code v2.1.154 或更新版本。
- 面向所有 paid plans，以及 Anthropic API、Amazon Bedrock、Google Cloud Vertex AI、Microsoft Foundry。
- Pro 计划需要在 `/config` 的 Dynamic workflows 行里开启。

### 2. 触发方式

`事实` 官方文档列出三种主要使用方式：

| 方式 | 含义 |
|---|---|
| `/deep-research <question>` | 使用 Claude Code 内置 workflow 做多来源 research、交叉核验和带引用报告。 |
| prompt 中包含 `workflow` | 让 Claude 为当前任务写并运行一个 workflow。 |
| `/effort ultracode` | 让 Claude 对每个 substantive task 自动规划 workflow。 |

`推断` `/effort ultracode` 很像把“是否需要编排”的判断交给 agent runtime。它适合高价值复杂任务，不适合日常小改动，因为官方也说明这种模式会消耗更多 tokens、耗时更长。

### 3. 运行和管理

`事实` workflow 在后台运行，主 session 保持可响应。用户可以用 `/workflows` 查看运行中和已完成的 workflow，并进入进度视图查看阶段、agent 数量、token 总量、耗时、单个 agent 的 prompt、近期工具调用和结果。

`事实` 运行中可以 pause / resume，可以 stop 某个 agent 或整个 workflow，可以 restart 某个 running agent。满意的 workflow script 可以保存为命令，位置包括：

- `.claude/workflows/`：项目级 workflow，可随仓库共享。
- `~/.claude/workflows/`：个人级 workflow，只对自己可见。

### 4. 执行约束

`事实` 官方文档明确列出一些约束：

| 约束 | 工程含义 |
|---|---|
| workflow 本身没有直接 filesystem 或 shell access | script 只负责编排，实际读写和命令由 agents 完成。 |
| workflow 运行中没有普通用户输入 | 需要人工签核的阶段应拆成多个 workflow，而不是一个 workflow 内等待人。 |
| 最多 16 个 concurrent agents | 控制本地资源使用。 |
| 每次 run 最多 1,000 个 agents | 防止 runaway loops。 |
| resume 只在同一 Claude Code session 内工作 | 退出 Claude Code 后，下一次需要 fresh run。 |

`推断` 这说明 Dynamic workflows 不是万能长期后台任务系统。它更像“单 session 内的大规模、可观察、可暂停的 agent 编排 run”。

### 5. 权限和成本

`事实` 每次 workflow 启动前通常会显示计划阶段，允许用户确认、查看 raw script、取消或记住授权。不同 permission mode 下提示频率不同。

`事实` workflow spawned subagents 会继承 tool allowlist，文件编辑自动批准；shell commands、web fetches、未 allowlist 的 MCP tools 仍可能在运行中要求权限。workflow 会产生明显更多 token 用量，并计入 plan usage 和 rate limits。

`推断` 对工程使用来说，Dynamic workflows 的默认安全策略是“启动前审查编排，运行中用 allowlist 限制工具”。如果要跑大型迁移，不能只看最终报告，应该先看 raw script 和阶段设计。

## 与 subagents、skills 的区别

`事实` 官方文档直接把 subagents、skills、workflows 放在同一张比较表里。核心差异是“谁持有计划”和“中间结果在哪里”。

整理如下：

| 机制 | 计划持有者 | 中间结果 | 可复用对象 | 典型规模 | 适合场景 |
|---|---|---|---|---|---|
| Subagents | 主会话里的 Claude 逐 turn 决定 | Claude context | worker 定义 | 少量 delegated tasks | 搜索、日志分析、局部调查、隔离上下文 |
| Skills | Claude 按 skill instructions 执行 | Claude context | 指令和 supporting files | 通常与 subagent 相近 | TDD、review、特定领域 procedure |
| Dynamic workflows | workflow script | script variables | 编排脚本本身 | 数十到数百 agents | 大范围 audit、迁移、交叉核验、多角度计划 |

`推断` skill 更像“操作纪律”，workflow 更像“执行编排”。例如：

```text
tdd-gate skill：
要求先 RED，再 GREEN，再 REFACTOR，并报告验证证据。

dynamic workflow：
把一个大型迁移拆成 N 个 phase，派出很多 subagents，
收集中间结果，再做交叉 review 或综合。
```

两者不是替代关系。复杂任务可以先由 writing-plans / tdd-gate 决定正确性边界，再由 dynamic workflow 执行大规模 fan-out 或交叉核验。

## 为什么它值得重视

### 1. 它把 context pressure 转移到 script state

`推断` coding agent 的一个长期瓶颈是上下文窗口。大量 subagent 返回的日志、文件摘要、发现和中间判断如果全部回到主对话，很快会污染上下文。Dynamic workflows 让中间结果留在 script variables，只把最终综合结果放回 session，这和“把状态外化为 artifacts”的 harness 思想一致。

### 2. 它让质量模式可重复

`事实` 官方文档提到 workflow 不只是运行更多 agents，也可以应用可重复的质量模式，例如让独立 agents 互相 adversarially review findings，或从多个角度起草计划再比较。

`推断` 这很重要。多数人使用 subagent 的方式只是“并行查资料”。Dynamic workflows 的价值是把“交叉核验、投票、对抗审查、阶段化汇总”写成可重复编排，而不是依赖主会话临时想起来。

### 3. 它把“任务编排”变成可保存资产

`事实` workflow run 可以保存为命令，之后出现在 slash command 自动补全里。

`推断` 这等于把一次成功的 agent 编排经验沉淀为团队资产。它和 skills、hooks、CLAUDE.md、AGENTS.md 是同一类东西：把人脑里的操作习惯外化，让之后的 agent run 能复用。

## 适用场景

更适合使用 Dynamic workflows 的任务：

| 场景 | 为什么适合 |
|---|---|
| codebase-wide bug sweep | 需要扫描很多模块，单会话容易遗漏和上下文膨胀。 |
| 500-file migration | 需要分片处理、汇总冲突、可能还要分阶段验证。 |
| cross-checked research | 需要多个来源互相核验，过滤未被支持的 claim。 |
| 多方案架构计划 | 可以让多个 agent 从不同角度起草，再比较 tradeoff。 |
| 安全、权限、数据流 audit | 适合分区域扫描，再做集中复核。 |

不适合默认使用的任务：

| 场景 | 为什么不适合 |
|---|---|
| 一两个文件的小改动 | workflow overhead 大于收益。 |
| 需求还没澄清 | 应先 brainstorming 或 spec review。 |
| 需要频繁人类决策 | workflow 中间不能普通用户输入，应拆阶段。 |
| 文件修改高度重叠 | 多 agents 容易冲突，应先 partition 或使用 worktrees。 |
| 没有验证方式的大改动 | 多 agents 只会扩大不确定性。 |

## 对 Coding Agent Harness Toolkit 的启发

### 1. 不要把 orchestration map 做成死流程

`个人整理框架` Dynamic workflows 反而强化了“轻量路由”的重要性。不是所有任务都该进 workflow；应该先判断任务层级：

```text
Tiny / Small：
直接执行 + final check。

Medium：
writing-plans + TDD / review gate。

Large：
orchestration map 选择是否需要 subagents / dynamic workflow / worktrees。
```

### 2. skill 和 workflow 要分层

`个人整理框架` 你的 Toolkit 里可以保持这个边界：

| 层 | 负责什么 |
|---|---|
| principles | 长期工程原则，例如 TDD、验证、上下文管理。 |
| workflows | 场景路线，例如 everyday-development、bugfix、orchestration-map。 |
| skills | 可触发的操作纪律，例如 tdd-gate、review-gate、writing-plans。 |
| future dynamic workflows | 大规模 agent 编排脚本，例如 repo audit、migration sweep、cross-check research。 |

也就是说，不需要把所有东西都升级成 skill。真正反复出现、且需要 runtime fan-out 的模式，才值得升级成 workflow script。

### 3. 大任务应该从“计划文档”升级到“可执行编排”

`个人整理框架` 现在的 `writing-plans` skill 产出的是人工可读 plan。Dynamic workflows 给出的方向是：当某类 plan 反复出现，并且步骤可以被稳定拆分，就可以继续演化为可运行编排。

演化路径可以是：

```text
一次性任务
-> checklist
-> prompt
-> skill
-> workflow script
-> runtime-managed workflow command
```

这和 `harness-design-log.md` 的设计反馈循环一致。不要因为看到 Dynamic workflows 就立刻复制一套复杂系统；先从真实失败和重复任务中抽象。

### 4. workflow 输出也要有 gate

`个人整理框架` Dynamic workflows 能提高覆盖面，但不能自动保证正确性。对大型 coding 任务，workflow 完成后仍需要：

- diff review。
- test / lint / build 证据。
- 对高风险模块做 human review。
- 对 workflow script 本身做审查，尤其是权限、文件范围、并发修改和验证策略。

可以把它看成：

```text
Dynamic workflow 扩大执行和发现能力；
review-gate / tdd-gate / verification-gate 保证最终放行质量。
```

## 与 Superpowers 的关系

`推断` Superpowers 和 Dynamic workflows 解决的是不同层面的问题：

| 维度 | Superpowers | Claude Code Dynamic workflows |
|---|---|---|
| 主要形态 | 一组 skill-like engineering gates | runtime-managed workflow scripts |
| 价值 | 把资深工程纪律做成触发点 | 把大规模 subagent 编排做成可运行脚本 |
| 状态位置 | 多数仍在主会话上下文和文件 artifact | script variables + runtime progress |
| 最强场景 | TDD、debugging、review、planning discipline | 大型 audit、migration、cross-check research |
| 风险 | 上下文吃紧、流程可能过重 | 成本高、权限和并发修改复杂 |

更好的组合不是二选一，而是：

```text
Superpowers-style gates 决定什么时候停、什么时候验证、什么时候 review；
Dynamic workflows 决定大型任务如何 fan out、交叉核验和汇总。
```

## 需要警惕的误解

### 误解一：dynamic workflow 等于所有任务都自动编排

不是。官方也强调适用场景是大任务、跨文件任务、需要大量 agents 或交叉核验的任务。routine work 应该回到普通 effort 或简单 workflow。

### 误解二：更多 agents 等于更可靠

不是。没有交叉核验、验证命令和 review gate，多 agents 只是更快地产生更多未经证实的结论。

### 误解三：workflow script 由 Claude 写，所以不用审查

不对。恰恰因为它会批量调度 agents、可能触发大量文件编辑和工具调用，启动前更应该看 planned phases 和 raw script。

### 误解四：Dynamic workflows 会替代 skills

不会。skills 适合封装操作纪律和领域 procedure；dynamic workflows 适合封装大规模编排。TDD、review、debugging 这类 gate 仍然适合做 skill。

## 对个人使用的建议

`个人整理框架` 如果你使用 Claude Code，可以按下面的规则尝试：

1. 先从 `/deep-research` 感受它的输出结构，观察它如何做 fan-out、source cross-check 和 synthesis。
2. 只在大任务 prompt 中显式写 `workflow`，不要一开始就打开 ultracode 常驻。
3. 第一次运行某类 workflow 前，选择查看 raw script，重点看阶段、agent 数、文件范围、验证步骤和权限风险。
4. 如果某个 workflow 跑得好，再保存为 `.claude/workflows/` 或 `~/.claude/workflows/`。
5. 对会修改代码的 workflow，优先配合 worktrees、测试命令和 review gate。
6. 记录一次 run 的失败模式：是拆分太粗、agent 数太多、验证不足、文件冲突，还是最终汇总不可信。再决定是否沉淀进 Toolkit。

如果当前工具不是 Claude Code，也可以借鉴思想，而不是照搬功能：

```text
用 plan 文档保存编排；
用 subagent 或多个 session 执行分片；
用 progress file 保存中间状态；
用 review-gate / verification-gate 收口；
用 harness-design-log 记录哪些模式值得自动化。
```

## 参考资料

- Anthropic / Claude Code Docs: [Orchestrate subagents at scale with dynamic workflows](https://code.claude.com/docs/en/workflows)
- Anthropic / Claude Code Docs: [Run agents in parallel](https://code.claude.com/docs/en/agents)
- Anthropic / Claude Code Docs: [Week 22 · May 25-29, 2026](https://code.claude.com/docs/en/whats-new/2026-w22)
- Anthropic Engineering: [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- Anthropic / Claude Code Docs: [Create custom subagents](https://code.claude.com/docs/en/sub-agents)
