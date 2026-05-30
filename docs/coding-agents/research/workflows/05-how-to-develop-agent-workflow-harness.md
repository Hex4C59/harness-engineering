# 如何培养 Agent Workflow Harness 设计能力

调研日期：2026-05-31

这篇文档回答一个问题：为什么有些人能创造出 Superpowers 这类 agent workflow harness，而大多数人只是“直接让 AI 开始做”？更重要的是：如何有意识地训练这种发现、抽象和沉淀能力。

本文基于公开互联网资料整理，尤其是 Anthropic 的 agent / Claude Code / long-running harness 文章，OpenAI Codex 和 eval 文档，以及 Superpowers 的公开 skill 源码。文中区分：

- `事实`：来源中明确表达的内容。
- `推断`：从多个来源共同模式中归纳出的结论。
- `个人整理框架`：面向本项目 `Coding Agent Harness Toolkit` 的落地方法。

## 核心结论

`推断` Agent 之所以叫 agent，是因为模型会动态参与规划、选择工具和调整路径；但可靠的 agent 使用，往往仍然需要外部 workflow harness 约束它在关键阶段停下来、给证据、接受 review。

这不是矛盾，而是分工：

```text
agent 负责动态判断和执行；
workflow harness 负责阶段边界、上下文入口、验证证据和失败回流。
```

Superpowers 的价值不在于“流程多”，而在于它把常见失败模式前面的拦截点写成了可复用 gates：

```text
需求不清 -> brainstorming
计划不可执行 -> writing-plans
先实现后补测试 -> test-driven-development
没复现就修 bug -> systematic-debugging
没验证就说完成 -> verification-before-completion
自己实现自己背书 -> code review / reviewer subagent
长上下文污染 -> subagent / worktree / progress artifacts
```

## 来源观察

### 1. Anthropic：workflow 和 agent 是两类东西，不要一上来就复杂化

`事实` Anthropic 在《Building effective agents》中区分了 workflow 和 agent：workflow 是 LLM 和工具沿预定义路径被编排；agent 是 LLM 动态控制自己的流程和工具使用。文章同时建议先找最简单可行方案，只有需要时再增加复杂度，因为 agentic system 会用延迟和成本换取更强任务表现。

对个人 coding agent 工作流的含义是：

```text
不要把 workflow 当成 agent 的替代品；
也不要把 agent 当成无需 workflow 的自动程序员。
```

`推断` 好的 coding agent harness 应该是风险分层的：小任务走短路径，高风险任务才升级到 brainstorming、planning、TDD、review、subagent。

### 2. Claude Code：上下文窗口是核心资源，验证必须变成可运行信号

`事实` Claude Code best practices 明确强调 context window 会很快填满，性能会随上下文膨胀而下降。它还建议给 Claude 一个可以验证工作的检查：测试、构建、截图对比、linter、fixture diff 等。没有可运行检查时，模型只能依赖“看起来完成”。

`事实` 同一份文档把最佳实践组织成类似阶段的建议：先探索，再计划，再编码；提供具体上下文；配置环境；写 `CLAUDE.md`；设置 permissions、hooks、skills、subagents。

`推断` 这说明高手工作流不是“更长 prompt”，而是把模型需要的上下文和放行条件变成外部 artifact：

```text
项目规则 -> AGENTS.md / CLAUDE.md
验证命令 -> scripts/check / test command
阶段流程 -> workflow / skill
完成证据 -> command output / screenshot / diff / review findings
```

### 3. Anthropic long-running harness：长任务失败来自“一次做太多”和“过早宣布完成”

`事实` Anthropic 在《Effective harnesses for long-running agents》中描述了长程 coding agent 的两个失败模式：模型倾向于一次做太多，导致上下文耗尽和半成品状态；后续 agent 看到已有进展后，又容易过早宣布整个任务完成。文章提出 initializer agent + coding agent 的两段式 harness：初始化环境、写 feature list、记录 progress、创建初始 git commit；后续 agent 每轮只推进一个 feature，并留下结构化更新。

`事实` 该文章还强调：feature list 中的功能起初都标为 failing；coding agent 只能改变 pass 状态，不能随意删除或改测试；每个新 session 开始时读取 progress、git log、feature list，并先跑基础端到端验证。

`推断` 这和 TDD / reviewable slice / project status 的思想高度一致：长任务需要的不是更强记忆，而是让每个新上下文能从稳定 artifact 接手。

### 4. Claude subagents：subagent 是上下文隔离工具，不是默认并行魔法

`事实` Claude Code subagents 文档说明：subagent 适合处理会淹没主会话的 side task，例如大量搜索结果、日志或文件内容；每个 subagent 有自己的上下文窗口、自定义 system prompt、工具权限和独立 permissions。

`推断` subagent 的核心价值不是“多线程更快”，而是：

- 主会话保持协调和决策。
- 高噪声探索隔离出去。
- reviewer 能用相对独立上下文检查实现。
- 工具权限可以按任务缩小。

因此，类似 Superpowers 的 `subagent-driven-development` 不能简单理解成“多派几个 agent”。它更像：

```text
主 agent 管计划和验收；
implementer subagent 做单个任务；
spec reviewer 检查是否符合计划；
quality reviewer 检查实现质量；
主 agent 不信任 subagent 自述，只看 diff、测试和 review 结果。
```

### 5. OpenAI Codex：先 Ask / plan，再 Code；prompt 要像 issue

`事实` OpenAI 的《How OpenAI uses Codex》提到，OpenAI 团队把 Codex 用于代码理解、重构迁移、性能优化、补测试、探索方案等。其 best practices 包括：大改动先用 Ask mode 生成 implementation plan，再切到 Code mode；把 prompt 写得像 GitHub issue，包含文件路径、组件名、diff、文档片段；用 `AGENTS.md` 提供持久项目上下文。

`推断` 这支持把 coding agent 工作拆成两个阶段：

```text
理解 / 计划阶段：Ask / brainstorming / writing-plans
执行阶段：Code / execute-plan-slice / tdd-gate
```

这也解释了为什么 `writing-plans` 这种 skill 有价值：它不是为了显得正式，而是为了让执行阶段不重新发明需求。

### 6. OpenAI eval 文档：复杂度升级应该由 eval 驱动

`事实` OpenAI evaluation best practices 认为 eval 是应对 AI 系统不确定性的结构化测试；建议 eval-driven development、早评估、写 scoped tests、记录日志、尽可能自动化，并用人类反馈校准自动评分。它还明确指出，随着架构从 single-turn 到 workflow、single-agent、multi-agent，复杂度和不确定性会增加；是否使用 multi-agent 应该由 eval 驱动，过早使用 multi-agent 会增加复杂度。

`推断` 这给 workflow harness 一个重要原则：

```text
不要因为流程看起来高级就升级复杂度；
只有当失败模式和评估证据说明需要更强 gate 时，才升级。
```

这也支持 `orchestration-map.md` 里的原则：使用最轻但足够可靠的流程。

### 7. Superpowers：把隐性工程纪律写成可触发 skills

`事实` Superpowers 的公开 skill 源码把流程拆成 `brainstorming`、`writing-plans`、`test-driven-development`、`subagent-driven-development` 等模块。`writing-plans` 要求把计划写成非常具体的执行脚本，包含文件路径、代码、命令和预期输出，并禁止 `TODO`、`TBD`、空泛的“处理边界情况”。`test-driven-development` 强制 RED-GREEN-REFACTOR。`subagent-driven-development` 要求每个任务派 fresh subagent，并经过 spec review 和 quality review。

`推断` Superpowers 的真正创新不是某个 prompt，而是把资深工程师的停止点和证据点做成了运行时可触发的 workflow gates。

## 如何训练自己发现 workflow harness

### 第一层：从 prompt 思维转向 failure-mode 思维

普通使用者的问题是：

```text
我该怎么让 AI 做得更好？
```

更强的问题是：

```text
AI 通常在哪里失败？
失败前有什么征兆？
我能不能在进入下一阶段前加一个 gate？
这个 gate 需要什么证据？
```

示例：

| 观察到的失败 | 抽象失败模式 | 可设计的 gate |
|---|---|---|
| agent 没问清需求就写代码 | 意图未冻结 | brainstorming |
| agent 计划写得像愿望清单 | 执行自由度太高 | writing-plans |
| agent 先实现再补测试 | 缺少 RED 证据 | tdd-gate |
| agent 修 bug 靠猜 | 根因未验证 | systematic debugging / bugfix workflow |
| agent 说完成但没跑验证 | 完成声明无证据 | final-check / verification gate |
| agent 改了太多文件 | diff 不可 review | reviewable-slices |
| reviewer 只听实现者总结 | 审查被叙述污染 | review-gate / independent reviewer |
| 长任务接不上 | 上下文交接失败 | project-status / progress log / plan status |

### 第二层：把一次失败沉淀成一个 gate

每次 agent 走偏后，做一个 5 行 postmortem：

```text
任务：
失败或险些失败的地方：
失败发生在哪个阶段：
下次进入下一阶段前应该要求什么证据：
这个证据适合写成 prompt、checklist、skill，还是 workflow：
```

如果同类问题出现 2-3 次，就值得沉淀。

### 第三层：用 Trigger / Input / Action / Evidence / Exit 描述 gate

一个可复用 gate 至少要写清：

| 字段 | 问题 |
|---|---|
| Trigger | 什么时候触发？ |
| Input | 要读哪些上下文和 artifact？ |
| Allowed Action | 允许做什么？ |
| Forbidden Action | 禁止做什么？ |
| Evidence | 通过 gate 需要什么证据？ |
| Exit | 通过后进入哪个阶段？ |
| Stop Condition | 什么情况必须停下来？ |

这套结构比“请认真一点”有效，因为它给 agent 明确边界。

### 第四层：先 checklist，再 prompt，再 skill

`个人整理框架` 不要一开始就写 skill。更稳的成长路径是：

```text
一次失败
-> checklist 条目
-> 可复制 prompt
-> 用真实任务验证 5-10 次
-> 触发条件稳定后升级为 skill
-> 多个 skill 之间再写 orchestration map
```

升级成 skill 的条件：

- 触发场景明确。
- 输入和输出稳定。
- 禁止事项明确。
- 失败时知道如何 stop。
- 跨项目复用价值高。
- 读入 skill 的上下文成本低于反复解释流程的成本。

### 第五层：用 eval / assert 判断 harness 是否真的变好

`事实` OpenAI eval 文档强调 eval-driven development、scoped tests、自动化、人类反馈校准和持续评估。

对 coding agent harness，可以把 eval 理解成 harness assert：

```text
这个 gate 是否真的减少了某类失败？
它是否让任务变慢但没有提升质量？
它是否误伤了小任务？
它失败时是否能诊断？
```

可用指标：

| 指标 | 含义 |
|---|---|
| escape rate | 这个 gate 应拦住但没拦住的问题比例 |
| false block rate | gate 误拦低风险任务的比例 |
| diagnostic value | 失败后能否定位原因 |
| runtime cost | 增加了多少时间和 token |
| reviewability | diff 是否更小、更容易审 |
| validation freshness | 完成声明是否有新鲜证据 |
| context load | 是否减少主会话污染 |

## 面向本项目的落地建议

`个人整理框架` 当前 `Coding Agent Harness Toolkit` 已经有：

- `orchestration-map.md`：轻量路由，不固定流水线。
- `brainstorming`：需求不清时澄清。
- `writing-plans`：把清楚需求写成 Execution Plan。
- `tdd-gate`：行为变化前强制 RED-GREEN-REFACTOR。
- `review-gate`：实现完成后只读审查 diff。
- `commit-gate`：提交边界和 message 建议。

下一步不应该继续堆 skill，而是增加一个 harness 设计反馈循环：

```text
docs/coding-agents/agent/playbooks/meta/harness-design-log.md
```

建议记录字段：

```text
日期：
任务：
agent 失败 / 险些失败：
失败阶段：
已有 gate 是否覆盖：
需要新增 / 修改的 gate：
先作为 checklist、prompt、skill 还是 workflow：
验证方式：
```

这样能避免“看到 Superpowers 很强，所以一口气复制所有流程”。真正应该复制的是它的演化方式：从失败模式抽象出 gate，再用真实任务验证。

## 反模式

### 反模式一：把 orchestration map 写成固定流水线

坏：

```text
所有任务都必须 brainstorming -> writing-plans -> tdd -> review -> commit
```

好：

```text
任务越小，流程越短；
风险越高，gate 越多；
复杂度升级由失败模式和验证证据驱动。
```

### 反模式二：把所有文档都升级成 skill

Skill 会进入 runtime 触发路径，成本比普通文档高。原则、背景和长解释不一定适合 skill。适合 skill 的是稳定、重复、边界清楚、有输出格式的 gate。

### 反模式三：只写流程，不写证据

“先计划、再实现、再验证”仍然太抽象。每个 gate 都应要求具体证据：

- 计划文件路径。
- 失败测试命令和失败原因。
- 通过命令和 exit code。
- diff 范围。
- review findings。
- 未运行验证和原因。

### 反模式四：过早 subagent 化

OpenAI eval 文档指出 multi-agent 会引入新的不确定性，是否使用 multi-agent 应由 eval 驱动。Claude Code 文档也把 subagent 定位为上下文隔离工具。不要因为 subagent 看起来高级就默认并行实现。

### 反模式五：用长上下文代替 artifact

长对话不是稳定交接。长任务需要：

- plan。
- project status。
- progress log。
- git history。
- feature / acceptance list。
- verification command。

这些 artifact 比“模型记得刚才聊过什么”可靠。

## 一句话总结

创造类似 Superpowers 的能力，不是写更神奇的 prompt，而是持续做这件事：

```text
观察 agent 失败模式
-> 找到失败前的拦截点
-> 定义可运行证据
-> 写成 checklist / prompt / skill
-> 用真实任务验证
-> 只在必要时升级编排复杂度
```

这就是从“使用 AI”进入“设计 AI 工作系统”的分水岭。

## 参考资料

- Anthropic Engineering: [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- Anthropic Engineering: [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- Anthropic Claude Code Docs: [Best practices for Claude Code](https://docs.anthropic.com/en/docs/claude-code/best-practices)
- Anthropic Claude Code Docs: [Create custom subagents](https://docs.anthropic.com/en/docs/claude-code/sub-agents)
- OpenAI: [How OpenAI uses Codex](https://openai.com/business/guides-and-resources/how-openai-uses-codex/)
- OpenAI API Docs: [Evaluation best practices](https://platform.openai.com/docs/guides/evaluation-best-practices)
- obra/superpowers: [brainstorming/SKILL.md](https://raw.githubusercontent.com/obra/superpowers/main/skills/brainstorming/SKILL.md)
- obra/superpowers: [writing-plans/SKILL.md](https://raw.githubusercontent.com/obra/superpowers/main/skills/writing-plans/SKILL.md)
- obra/superpowers: [test-driven-development/SKILL.md](https://raw.githubusercontent.com/obra/superpowers/main/skills/test-driven-development/SKILL.md)
- obra/superpowers: [subagent-driven-development/SKILL.md](https://raw.githubusercontent.com/obra/superpowers/main/skills/subagent-driven-development/SKILL.md)
