# AI 辅助代码 Review：更快，但不能只靠 AI

调研日期：2026-05-30

## 这篇文档回答什么

Coding agent 写代码越来越快以后，瓶颈会自然转移到 review：

```text
过去：人类写代码慢，review 跟得上。
现在：agent 生成代码快，review 成为主要队列。
未来一段时间：真正稀缺的是可靠判断、上下文理解和验收能力。
```

这篇文档回答：

1. 有没有比纯人工更快的 code review 方法？
2. AI 能不能 review AI 写的代码？
3. 如果用 AI review，可靠性从哪里来？
4. 应该让 AI 看什么、不看什么？
5. 个人 coding agent 工作流和团队 PR 流程分别怎么落地？

核心结论：

> AI review 可以显著提速，但不能把 `LGTM` 外包给模型。可靠的方法不是“让 AI 代替 reviewer”，而是建立一个 **AI 辅助 review harness**：小变更、明确规格、确定性检查、独立 AI reviewer、专项 subagent、人类风险分层复核、反馈校准。AI 负责压缩上下文、提前发现低到中风险问题、生成检查清单和指出测试缺口；人类负责最终取舍、业务语义、架构边界、安全责任和是否合并。

一句话操作答案：

```text
先让写代码的 agent 自审。
再让独立上下文的 AI reviewer 审 diff。
确定性工具必须先过。
人类只深审高风险点和 AI 发现。
所有 AI 评论必须有文件位置、证据、失败场景和验证方式。
```

## 来源说明

本文优先使用论文、官方文档、工程实践文档和资深工程师原文。AI code review 工具变化很快，涉及具体产品能力时以 2026-05-30 可查公开资料为准。

| 来源 | 类型 | 主要价值 |
|---|---|---|
| [Modern Code Review: A Case Study at Google](https://research.google/pubs/modern-code-review-a-case-study-at-google/) | Google 代码评审研究 | 说明现代 code review 的目标、成本和组织实践 |
| [Google Engineering Practices: What to look for in a code review](https://google.github.io/eng-practices/review/reviewer/looking-for.html) | 官方工程实践 | 给出人工 review 应覆盖的设计、功能、复杂度、测试、命名、文档、上下文等维度 |
| [Google Engineering Practices: Speed of Code Reviews](https://google.github.io/eng-practices/review/reviewer/speed.html) | 官方工程实践 | 说明快 review 的关键是快速响应、小 CL、不要牺牲标准 |
| [AI-Assisted Assessment of Coding Practices in Modern Code Review](https://arxiv.org/html/2405.13565v1) | Google / AIware 2024 论文 | AutoCommenter 在 Google 大规模部署的经验：高精度优先、分阶段 rollout、用户反馈和 suppression 机制 |
| [Finding GPT-4's mistakes with GPT-4](https://openai.com/index/finding-gpt4s-mistakes-with-gpt-4/) / [LLM Critics Help Catch LLM Bugs](https://arxiv.org/abs/2407.00215) | OpenAI 研究 | CriticGPT 表明“人类 + AI critic”比单独人类更容易发现模型输出错误，但 AI critic 也会出错 |
| [Responsible use of GitHub Copilot code review](https://docs.github.com/en/copilot/responsible-use/code-review) | GitHub 官方文档 | 明确 Copilot code review 用于补充而非替代人类 review，并列出漏报、误报、不准确或不安全建议等限制 |
| [About GitHub Copilot code review](https://docs.github.com/en/copilot/concepts/agents/code-review) | GitHub 官方文档 | 说明 Copilot code review 的上下文、custom instructions、agentic capabilities、验证要求 |
| [Rethinking Code Review Workflows with LLM Assistance](https://arxiv.org/html/2505.16339v1) | 2025 实证研究 | 研究 AI-led summary 和 on-demand assistant 两种模式，指出大 PR / 陌生代码下 AI 摘要有价值，但信任、误报、延迟和集成摩擦仍是问题 |
| [Automating Code Review Activities by Large-Scale Pre-training](https://arxiv.org/abs/2203.09095) | FSE 2022 论文 | CodeReviewer 代表早期自动化 code review 研究，覆盖代码质量估计、评论生成、代码修复等任务 |
| [Thoughts on coding agents](https://dennybritz.com/posts/coding-agents) | 资深工程师经验 | 指出 coding agent 的真正瓶颈常常是上下文、反馈和人类判断 |
| [Embracing the parallel coding agent lifestyle](https://simonwillison.net/2025/Oct/5/parallel-coding-agents/) | 资深开发者经验 | 强调 review 是并行 agent 的自然瓶颈，规格越清楚，review 越省力 |
| [Superpowers: How I'm using coding agents in October 2025](https://blog.fsck.com/2025/10/09/superpowers) | coding agent 工作流 | 展示 brainstorm、plan、TDD、subagent 实现、code review skill 的系统化方式 |

## 先给操作答案

推荐把 AI review 放进一个三段式流程。

### 第一段：作者侧 AI 自审

写代码的 agent 完成实现后，先不要让人类看 diff，而是让它执行一轮作者侧自审：

```text
目标：在提交给人类之前，先清掉低级错误、测试缺口、无关修改、过度设计和明显回归。
输入：任务目标、非目标、实现计划、diff、测试输出。
输出：自审报告、已修复问题、未修复风险、需要人类判断的问题。
```

这一段不够独立，不能当最终 review，因为它会受到自己实现路径的锚定影响。但它非常适合减少人类看到的噪声。

### 第二段：独立 AI reviewer 审 diff

再开一个新会话、subagent、PR bot 或不同模型，让它只扮演 reviewer：

```text
目标：用新鲜视角审查 diff。
输入：PR 描述、AGENTS.md / CLAUDE.md / copilot instructions、相关 README、diff、测试输出。
限制：默认不改代码，只输出 findings。
输出：按严重程度排序的问题，每个问题必须有文件位置、证据、失败场景、建议修复和验证方式。
```

这一段是 AI review 的核心。它最好与实现会话隔离，避免“我刚写完所以我觉得没问题”的路径依赖。

### 第三段：人类风险分层复核

人类 reviewer 不再从零开始读所有内容，而是按风险分层：

| 变更类型 | 人类 review 深度 | AI review 角色 |
|---|---|---|
| 文档、注释、简单测试补充 | 抽查 | 检查准确性、断链、术语一致性 |
| 小 bugfix，有回归测试 | 看测试是否真的覆盖 bug，再抽读核心 diff | 找边界条件和测试缺口 |
| 普通功能，小到中等 PR | 读 PR 摘要、核心路径、AI findings、测试证据 | 总结变更、找明显逻辑问题 |
| 架构、权限、认证、计费、数据删除、迁移、并发、安全 | 必须人工深审 | AI 只能做辅助清单和专项提示 |
| 大重构或超大 PR | 先要求拆分；不能拆时分阶段 review | 生成文件地图和风险地图 |

关键是：

```text
AI 给你缩短进入状态的时间。
AI 帮你指出候选问题。
AI 不能替你承担最终责任。
```

## 为什么 review 会成为瓶颈

人工 code review 本来就不是“看一眼代码”。Google 的工程实践把 review 拆成很多维度：设计是否合理、功能是否符合用户需要、复杂度是否过高、测试是否有效、命名是否清楚、注释是否解释了为什么、文档是否同步、风格是否一致、是否理解每一行、是否看了上下文。

在 coding agent 出现前，写代码本身限制了变更速度。现在 agent 可以在几分钟内生成几百行改动，甚至并行生成多个方案。于是瓶颈转移到了：

1. 人类能不能快速理解变更意图。
2. 人类能不能判断实现是否符合长期架构。
3. 人类能不能找出 agent 没有意识到的边界条件。
4. 人类能不能验证测试不是“看起来覆盖了”。
5. 人类能不能抵抗“大 diff 已经跑通了”的合并压力。

Simon Willison 在并行 coding agent 经验里提到一个很现实的限制：AI 生成代码需要 review，而 review 速度自然限制了并行收益。Denny Britz 也强调，coding agent 并没有消灭软件工程，只是把工作从手写代码转移到规格、上下文、反馈和判断。

所以更快 review 的第一原则不是“让 AI 多写几条评论”，而是减少人类需要从零理解的东西。

## 什么叫可靠的 AI review

这里的可靠不是指“AI 能发现所有 bug”。这在目前不现实。

更合理的定义是：

```text
在给定成本和时间内，AI review 能稳定降低人类漏看重要问题的概率，
同时把误报、噪声和错误修复建议控制在可接受范围内。
```

可靠性来自系统设计，而不是单个模型回答。

### 可靠性的五个来源

| 来源 | 作用 | 例子 |
|---|---|---|
| 小变更 | 降低理解成本和漏审概率 | 大 PR 先拆成小 PR |
| 明确规格 | 让 reviewer 知道代码应该满足什么 | PR 里写目标、非目标、验收标准 |
| 确定性检查 | 用编译器、类型、lint、测试兜住可机器判断的问题 | CI、单测、类型检查、静态扫描 |
| 独立 AI reviewer | 用新上下文发现实现者没看到的问题 | reviewer 会话不继承实现会话 |
| 人类风险复核 | 处理业务语义、架构、安全和责任问题 | 高风险模块必须 human approval |

这和 OpenAI CriticGPT 的启发一致：AI critic 最有价值的位置不是单独裁判，而是帮助人类发现模型输出中的问题。OpenAI 报告中，人类在 CriticGPT 帮助下 review ChatGPT 代码输出，表现优于无帮助人类的比例超过 60%；同时他们也强调 CriticGPT 的建议并不总是正确，模型单独工作会有幻觉问题。

GitHub Copilot code review 官方文档也给出类似边界：它可以快速反馈代码，但应补充而不是替代人类 review；它可能漏掉问题，可能产生误报，也可能给出不准确或不安全的代码建议。

## AI review 擅长什么

AI reviewer 擅长的不是“最终合并判断”，而是下面这些高频、结构化、可解释的辅助任务。

### 1. PR 摘要和风险地图

对大 PR 或陌生代码，AI 可以先输出：

- 改了哪些文件。
- 每类文件承担什么角色。
- 变更的主要行为路径。
- 哪些地方最值得人类看。
- 哪些测试覆盖了变更。
- 哪些风险没有测试证据。

2025 年的 `Rethinking Code Review Workflows with LLM Assistance` 研究发现，开发者通常重视 AI 生成的摘要和上下文解释，尤其是在大 PR 或不熟悉的代码上。研究也提醒，AI 工具必须嵌入现有环境、输出简洁可执行、延迟低，并支持主动摘要和按需问答两种模式。

### 2. 规则和最佳实践检查

Google 的 AutoCommenter 论文很有代表性。它不是试图完全替代 reviewer，而是针对“编码最佳实践”自动生成评论，例如 C++、Java、Python、Go 的语言习惯和内部规范。Google 的经验说明：

- 很多 best practice 过去只能靠资深 reviewer 反复指出。
- LLM 可以学习这些高频 review 模式。
- 工业部署必须高精度优先，宁愿少报也不要刷屏。
- 必须有分阶段 rollout、阈值、suppression、用户反馈和持续校准。

这给个人和团队一个重要启发：

```text
AI review 最好先从稳定规则开始，而不是一上来挑战最难的架构判断。
```

### 3. 测试缺口发现

AI 很适合问：

- 这次改动有没有对应测试？
- 测试是否真的会在代码坏掉时失败？
- 是否只测 happy path？
- 边界值、错误输入、权限失败、并发、迁移失败是否覆盖？
- 有没有快照测试掩盖真实语义？

Google 工程实践也提醒：测试本身也需要 review。测试不会自动证明自己有效，必须有人判断它是否真的有用。

### 4. 边界条件和失败路径枚举

AI 可以快速枚举：

- 空输入、重复输入、非法输入。
- 超时、取消、重试。
- 权限不足。
- 旧数据、脏数据、迁移中状态。
- 并发写、重复提交、幂等性。
- 上游 API 失败或返回异常格式。

这类任务很适合 AI，因为它像一个不知疲倦的 checklist generator。但最后哪些边界真正重要，仍然要人类结合业务判断。

### 5. 机械一致性检查

例如：

- 新字段是否同步到序列化、校验、文档、测试 fixture。
- 新 API 是否更新 README、OpenAPI、SDK、示例。
- 删除代码是否还有引用。
- 错误码、日志字段、metrics 名称是否一致。
- 配置变更是否更新部署说明。

这些问题不一定深，但很容易漏。AI reviewer 可以显著降低遗漏率。

### 6. 专项 review

AI 可以作为 subagent 做专项检查：

- security reviewer。
- test reviewer。
- performance reviewer。
- migration reviewer。
- accessibility reviewer。
- frontend UX reviewer。
- API compatibility reviewer。

专项 reviewer 的好处是 prompt 更聚焦，输出更少，更容易校准。它不需要看所有东西，只需要围绕一个风险维度审查。

## AI review 不擅长什么

下面这些场景不能指望 AI 独立判断。

### 1. 业务语义是否正确

如果需求本身没有写清楚，AI reviewer 很可能只能判断“代码看起来合理”。它不知道真实业务意图、产品取舍、历史事故和团队隐性规则。

解决方式不是让 AI 猜，而是把 review 输入补齐：

```text
目标是什么？
非目标是什么？
哪些行为不能改变？
哪些兼容性必须保留？
哪些用户或系统会受影响？
```

### 2. 架构长期成本

AI 能指出复杂度和重复，但很难独立判断：

- 这层抽象是否值得。
- 这个依赖是否会锁死未来路线。
- 这个模块边界是否符合团队长期规划。
- 这个看似小的例外是否会成为惯例。

这仍然是资深工程师的工作。

### 3. 安全、权限、隐私和合规

AI 可以帮你找 SQL 注入、XSS、权限绕过、敏感日志等模式，但不能作为最终安全审计。尤其是认证、授权、计费、密钥、加密、数据导出、数据删除、审计日志、隐私字段等场景，必须有人工和专业工具复核。

### 4. 并发和分布式系统正确性

死锁、竞态、幂等、消息重复、事务边界、缓存一致性、时钟问题，很多不是看一段 diff 就能证明的。AI 可以列问题，但不能代替设计推演、压力测试和故障演练。

### 5. 生成修复建议的正确性

GitHub 官方文档明确提醒：Copilot code review 的代码建议可能看似有效，但语义、语法或安全性未必正确。这个提醒适用于所有 AI reviewer。

一个实用规则：

```text
AI 可以提出修复方向。
AI 生成的修复必须重新测试。
高风险修复必须作为新的 diff 被再次 review。
```

## 推荐方法：AI 辅助 Review Harness

把 AI review 当成一个 harness，而不是一条 prompt。

### Review harness 的输入

每个 PR 至少应该提供：

```text
1. 变更目标：这次解决什么。
2. 非目标：明确不做什么。
3. 风险说明：可能影响哪些模块、用户、数据或接口。
4. 测试证据：跑了哪些命令，结果是什么。
5. 相关上下文：issue、设计文档、README、AGENTS.md、架构约定。
6. diff：最好是小 diff；大 diff 要先拆。
```

没有这些输入，AI reviewer 会更容易幻觉，也更容易给出泛泛建议。

### Review harness 的输出契约

要求 AI reviewer 只输出结构化 findings：

```text
Severity: Critical / High / Medium / Low / Nit
Location: path:line
Problem: 具体问题
Evidence: 从 diff 或上下文看到的证据
Failure scenario: 真实失败场景
Suggested fix: 最小修复方向
Verification: 如何验证修复有效
Confidence: High / Medium / Low
```

不要让 AI 输出长篇泛泛总结。没有文件位置和失败场景的问题，默认不进入 review 队列。

### Review harness 的执行顺序

推荐顺序：

```text
1. 作者侧 agent 自审。
2. 格式化、lint、类型检查、单测、集成测试、静态扫描。
3. 独立 AI reviewer 审 diff。
4. 高风险维度派专项 reviewer。
5. 人类 reviewer 看 PR 摘要、风险地图、AI findings、测试证据和核心 diff。
6. 修复后再次跑检查，并让 AI 做 patch verification。
```

注意顺序：确定性工具应该尽早跑。不要让 AI 浪费时间指出 formatter、lint、编译错误这类机器能直接判断的问题。

## Prompt 模板：作者侧自审

适用于 agent 完成代码后，在提交给人类前自查。

```text
你现在不是实现者，而是提交作者的第一轮自审 reviewer。

请基于以下输入审查当前改动：
- 原始任务目标和非目标
- 实现计划
- 当前 diff
- 测试输出
- 项目约定，例如 AGENTS.md、README、测试约定

要求：
1. 先判断实现是否偏离任务目标或扩大范围。
2. 找出可能导致 bug、回归、数据错误、安全问题、兼容性问题的地方。
3. 检查测试是否真的覆盖关键行为，尤其是失败路径和边界条件。
4. 检查是否有无关修改、过度抽象、重复代码、命名不清、文档遗漏。
5. 不要夸奖，不要写泛泛建议。
6. 每个问题必须包含文件位置、证据、失败场景、最小修复建议和验证方式。
7. 如果没有发现问题，输出“未发现必须修复的问题”，并列出剩余风险。

输出格式：

## Must Fix
- [Severity] path:line
  Problem:
  Evidence:
  Failure scenario:
  Suggested fix:
  Verification:

## Should Fix

## Test Gaps

## Questions For Human

## Remaining Risk
```

如果你希望 agent 自审后直接修，可以加一句：

```text
先只修 Must Fix 和明显测试缺口。不要处理 Nit，不要做无关重构。修完后重新运行验证命令并报告结果。
```

## Prompt 模板：独立 AI reviewer

适用于新会话、subagent、Codex review、Claude Code reviewer、Copilot Chat、PR bot。

```text
你是一个独立 code reviewer。你的任务是审查当前 diff，不要修改代码。

请先阅读：
- AGENTS.md 或项目协作规则
- README.md 和相关模块 README
- PR 描述或任务说明
- 测试输出
- 当前 diff

Review 重点按优先级排序：
1. 会导致错误行为、数据损坏、安全问题、权限绕过、兼容性破坏的 bug。
2. 缺失或无效的测试。
3. 复杂度、过度设计、长期维护风险。
4. 文档、迁移、配置、API 合约遗漏。

限制：
- 不要评论已经由 formatter/linter 自动处理的问题。
- 不要输出泛泛的“建议考虑”。
- 不要假设不存在的需求。如果上下文不足，请列为 Question。
- 最多输出 10 个 findings，按严重程度排序。
- 每个 finding 必须有 path:line、证据、失败场景、最小修复建议和验证方式。
- 如果你不确定，请标明 Confidence: Low，不要装作确定。

输出：
## Findings
1. Severity:
   Location:
   Problem:
   Evidence:
   Failure scenario:
   Suggested fix:
   Verification:
   Confidence:

## Test Gaps

## Questions

## Overall Risk
```

这个 prompt 的重点是把 AI 从“写作文模式”拉回“review findings 模式”。

## Prompt 模板：安全专项 reviewer

适用于认证、授权、输入处理、文件上传、支付、数据导出、日志、密钥、加密、依赖升级等变更。

```text
你是 security-focused code reviewer。只审查安全和隐私风险，不要修改代码。

请重点检查：
1. 认证和授权是否可绕过。
2. 用户输入是否进入 SQL、shell、模板、HTML、文件路径、URL、日志。
3. 是否新增敏感数据暴露、日志泄漏、错误信息泄漏。
4. 是否破坏 CSRF、CORS、rate limit、tenant isolation。
5. 是否引入不安全默认值、弱加密、密钥硬编码。
6. 是否缺少安全相关测试。

输出只包含可证据化的 findings。每个 finding 必须包含：
- path:line
- 攻击或滥用场景
- 受影响资产
- 最小修复建议
- 验证方式
- confidence

不要把普通代码风格问题放进输出。
```

## Prompt 模板：测试专项 reviewer

适用于你怀疑“测试写了但不可靠”的 PR。

```text
你是 test reviewer。请只审查测试质量，不要修改代码。

请检查：
1. 新增或修改的测试是否会在实现错误时失败。
2. 是否覆盖真实需求，而不是覆盖当前实现细节。
3. 是否有 happy path 之外的失败路径、边界值、权限失败、并发或兼容场景。
4. 是否有脆弱测试、过度 mock、无意义断言、只检查 snapshot 的问题。
5. 是否需要更低层或更高层测试补充。

输出：
## Missing Tests
## Weak Tests
## Over-specified Tests
## Suggested Minimal Test Cases

每条建议必须说明它能防住什么具体 bug。
```

## Prompt 模板：Patch verification

修复 AI 或人类指出的问题后，再让 AI 做一次“补丁验证”，它只看新增修复是否真的解决问题。

```text
你是 patch verification reviewer。

请比较上一轮 findings 和当前 diff，判断：
1. 每个 Must Fix 是否已经被解决。
2. 修复是否引入新问题。
3. 测试是否覆盖了原失败场景。
4. 是否还有未解决风险。

输出表格：
| Finding | Status | Evidence | Remaining risk |

Status 只能是：
- Resolved
- Partially resolved
- Not resolved
- Cannot verify

不要提出新的大范围重构建议，除非新问题会导致严重 bug。
```

## 人类 reviewer 应该怎么用 AI 输出

AI 输出不是 verdict，而是 review queue。

推荐处理方式：

```text
1. 先读 PR 目标和非目标。
2. 看 AI 的 diff summary，但不要完全依赖它。
3. 看 AI 标出的 High / Critical findings。
4. 对每条 finding 做 triage：Accept / Reject / Need more context。
5. 对高风险文件自己读核心逻辑。
6. 看测试是否能证明验收标准。
7. 决定是否需要第二个专项 AI reviewer 或人类专家。
8. 最终由人类给出 approve / request changes。
```

一个好习惯是要求 AI reviewer 给出 `Overall Risk`，但人类必须自己决定是否接受这个风险评级。AI 经常低估业务风险，也可能高估普通风格问题。

## AI review 的分层策略

不要所有 PR 都用同样重的流程。可以按风险分层。

### Level 0：纯确定性检查

适合：

- formatter 变更。
- typo。
- 注释修正。
- 生成文件更新。

流程：

```text
formatter / lint / tests pass
人类抽查
```

### Level 1：AI pre-review + 人类抽查

适合：

- 小 bugfix。
- 小测试补充。
- 文档同步。
- 局部重命名。

流程：

```text
agent 自审
确定性检查
AI reviewer
人类看 findings 和核心 diff
```

### Level 2：独立 AI reviewer + 专项 reviewer + 人类 review

适合：

- 普通功能。
- 多文件改动。
- 中等复杂度重构。
- API 行为变化。

流程：

```text
PR spec
测试证据
独立 AI reviewer
测试专项 reviewer
人类深读核心路径
```

### Level 3：高风险人工主导

适合：

- auth / permission。
- payment / billing。
- cryptography / key management。
- data deletion / migration。
- privacy / compliance。
- concurrency / distributed systems。
- production incident hotfix。
- public API breaking change。

流程：

```text
设计 review
安全或领域专家 review
专项 AI reviewer 只作辅助
更严格测试和回滚计划
人工 approval
```

AI 在 Level 3 的价值是“帮人类生成检查清单和候选问题”，不是“决定安全”。

## 如何提高 AI review 的准确性

### 1. 缩小 diff

Google Engineering Practices 对大 CL 的建议很直接：如果变更太大，通常应该要求拆分。AI 时代这个原则更重要。大 diff 不只是人类难审，AI 也更容易漏掉关键细节。

推荐限制：

```text
一个 PR 只做一件事。
格式化和功能改动分开。
重命名和行为改动分开。
迁移脚本和业务使用分开。
测试重构和产品逻辑分开。
```

### 2. 让 PR 自带规格

AI reviewer 需要知道“应该是什么”。没有规格，它只能审“代码像不像能跑”。

推荐 PR 模板：

```text
## Goal

## Non-goals

## User-visible behavior

## Risk areas

## Test evidence

## Rollback plan

## Notes for reviewers
```

### 3. 给 AI repository rules

GitHub Copilot code review 支持 custom instructions；Claude / Codex / 其他 CLI agent 通常也能读取 `AGENTS.md`、`CLAUDE.md` 或项目文档。

适合写进规则的内容：

- 测试命令。
- 架构边界。
- 不允许引入的依赖。
- 错误处理约定。
- 日志和 metrics 约定。
- 安全敏感模块。
- review 输出格式。
- 哪些问题属于 blocking。

不要写太空泛的规则，例如“写高质量代码”。要写可执行、可检查的规则。

### 4. 要求证据化输出

AI review 最大的问题之一是看起来很自信。解决方法是强制它给证据。

差的评论：

```text
这里可能有性能问题，建议优化。
```

好的评论：

```text
Severity: High
Location: src/search/index.ts:142
Problem: 每次请求都会重新扫描所有 documents，绕过了现有 cache。
Evidence: 新增的 searchDocuments() 在 handler 内直接读取 full corpus，没有使用 existing getSearchIndex()。
Failure scenario: 10k documents 时，每个 search request 都做 O(n) JSON parse，会导致 p95 latency 上升。
Suggested fix: 复用 getSearchIndex()，并在更新 document 后 invalidate cache。
Verification: 增加测试覆盖重复搜索只构建一次 index，并运行 search benchmark。
```

### 5. 限制输出数量

很多 AI reviewer 会输出一堆中低价值建议，反而增加人类负担。推荐：

```text
最多 10 条 findings。
Critical / High 优先。
Nit 默认不输出，除非会影响理解或一致性。
```

### 6. 使用独立上下文

作者 agent 自审有价值，但容易自我确认。真正的 AI reviewer 最好是：

- 新会话。
- 不继承实现过程。
- 只看 PR 描述、项目规则、diff、测试输出。
- 不默认相信作者总结。

如果工具支持 subagent，这是一个非常适合 subagent 的场景。主 agent 负责任务和合成，review subagent 负责专项审查。

### 7. 高风险问题用多视角

对安全、并发、迁移，可以让两个不同 prompt 或不同模型分别看：

```text
一个 security reviewer。
一个 test reviewer。
一个 architecture reviewer。
```

但不要无限并行。并行 reviewer 会增加 review 债。每多一个 reviewer，就多一份需要人类 triage 的输出。

## 如何校准 AI reviewer

可靠 AI review 必须校准，不校准就会慢慢变成噪声源。

### 需要记录的指标

| 指标 | 说明 |
|---|---|
| AI comment acceptance rate | AI 评论中有多少被人类接受或导致代码修改 |
| false positive rate | AI 评论中有多少被明确驳回 |
| missed issue rate | 人类或线上事故发现但 AI 漏掉的问题 |
| time to first review | AI 是否缩短首次反馈时间 |
| human review time | 人类实际 review 时间是否下降 |
| PR cycle time | 从 ready for review 到 merge 的时间 |
| high-risk coverage | 高风险 PR 是否仍有人工专家 review |
| fix verification accuracy | AI 判断“已修复”的准确率 |

Google AutoCommenter 的经验说明，用户反馈和 suppression 很重要。哪怕少量糟糕体验也会损害信任。个人工作流里可以不做复杂系统，但至少要记录：

```text
哪些 AI review 评论有用？
哪些经常误报？
哪些规则应该写进 AGENTS.md？
哪些问题 AI 总是漏？
```

### 反馈标签

团队可以给 AI comments 打简单标签：

```text
accepted
rejected-false-positive
rejected-not-important
needs-human-context
fixed-by-author
missed-by-ai
```

几周后回看，就能知道 AI reviewer 适合你们项目的哪些类别。

### 阈值策略

AutoCommenter 的一个关键经验是高精度优先。个人和团队也可以采用类似策略：

```text
宁愿 AI 少报，也不要每天刷出一堆低价值评论。
只让 AI 输出它能证据化的问题。
低置信度问题进入 Questions，不进入 Findings。
```

## 个人工作流：Codex / Claude / CLI agent 怎么用

如果你是单人开发或小团队，可以这样落地。

### 1. 写代码前

让 agent 先生成短规格：

```text
请先不要写代码。请阅读相关文件后输出：
1. 目标
2. 非目标
3. 修改计划
4. 风险点
5. 测试计划
等我确认后再实现。
```

这一步会显著降低后续 review 成本，因为你 review 的是“是否按你确认过的计划实现”。

### 2. 写代码后

让实现 agent 自审：

```text
请按作者侧自审模板 review 当前 diff。先不要修改代码，先列 Must Fix、Should Fix、Test Gaps 和 Questions。
```

然后让它只修 Must Fix 和测试缺口。

### 3. 开新会话做独立 review

新开 Codex / Claude 会话，给它：

- `AGENTS.md`
- 相关 README
- 任务说明
- diff
- 测试输出

让它按“独立 AI reviewer”模板输出 findings。

### 4. 人类最后看

你自己重点看：

- PR 目标是否正确。
- diff 是否局部且必要。
- AI 标出的 High / Critical。
- 测试是否真的覆盖行为。
- 是否有架构、安全、数据风险。

这比从零读完整 diff 快很多，但仍然保留了最终判断。

## 团队工作流：PR Bot 怎么接入

团队可以从低风险环节开始，不要一上来强制 AI bot 阻塞所有 PR。

### 第 1 周：只做 AI 摘要

Bot 只输出：

- 变更摘要。
- 文件地图。
- 风险区域。
- 测试证据摘要。
- reviewer 建议关注点。

不输出 blocking comments。目标是降低进入状态成本。

### 第 2 周：加入非阻塞 findings

Bot 输出结构化 findings，但默认不阻塞 merge。人类 reviewer 给标签：

```text
useful
false-positive
not-worth-commenting
missed-important-issue
```

### 第 3 周：低风险规则自动化

把高接受率规则变成：

- lint / static analysis。
- repo-specific script。
- AI reviewer 固定检查项。
- PR 模板要求。

能确定性检查的，不要长期留给 LLM。

### 第 4 周：风险分层

建立策略：

```text
低风险 PR：AI review + 人类抽查。
普通 PR：AI review + 人类 review。
高风险 PR：专项 AI review + 专家人工 review。
超大 PR：要求拆分或设计 review。
```

这样 AI review 才不会变成“所有 PR 都多一个嘈杂 reviewer”。

## AGENTS.md 可复用片段

可以把下面这段放进项目的 `AGENTS.md` 或 `docs/development.md`。

```markdown
## AI-assisted Code Review

Before asking a human to review agent-generated code:

1. Run formatter, lint, type checks, and relevant tests.
2. Perform an author-side self-review of the diff.
3. Fix only Must Fix issues and clear test gaps.
4. Record test evidence in the PR description.

For AI review:

- Use a fresh context whenever possible.
- The reviewer must not modify code unless explicitly asked.
- Findings must include severity, file:line, evidence, failure scenario, minimal fix, and verification.
- Do not comment on formatter/linter issues.
- Do not output vague suggestions without concrete failure scenarios.
- Limit findings to the top 10 by severity.
- Put low-confidence concerns under Questions.

Human approval is still required for:

- auth, permissions, payments, security, privacy, data deletion, migrations, public API changes, concurrency, and production incident fixes.
- large or cross-cutting refactors.
- changes where business intent is unclear.
```

## 常见失败模式和对策

| 失败模式 | 表现 | 对策 |
|---|---|---|
| AI 误报太多 | reviewer 开始忽略 bot | 限制 findings 数量；要求失败场景；低置信度进 Questions |
| AI 漏掉关键问题 | 看起来 review 过但仍出事故 | 高风险 PR 必须人工深审；引入专项 reviewer；补测试 |
| AI 幻觉项目规则 | 引用不存在的约定 | 要求引用文件和行；上下文不足时必须写 Question |
| AI summary 误导人类 | 人类只看摘要不看核心 diff | 人类必须看风险文件和测试证据 |
| AI 修复引入新 bug | suggested change 被直接接受 | 所有 AI 修复重新跑测试并二次 review |
| 大 PR 让 AI 和人都失效 | findings 泛泛、漏审多 | 拆 PR；先让 AI 生成拆分建议 |
| review 变慢 | 多个 AI reviewer 输出太多 | 只对高风险维度派专项 reviewer；普通 PR 保持一轮 |
| 作者 prompt 太模糊 | AI reviewer 不知道验收标准 | PR 模板强制目标、非目标、风险、测试证据 |

## AI reviewer 的边界话术

如果你要在团队里推广，最好提前统一措辞：

```text
AI reviewer is a co-reviewer, not an approver.
AI comments are leads, not verdicts.
Human reviewers remain responsible for merge decisions.
Deterministic checks beat model opinions.
High-risk changes require human expert review.
```

中文可以写成：

```text
AI reviewer 是协审，不是批准人。
AI 评论是线索，不是结论。
合并责任仍在人类 reviewer。
能用确定性工具检查的，不交给模型判断。
高风险变更必须有人类专家 review。
```

## 和现有 coding agent 工作流的关系

这篇文档可以和前几篇一起看：

- [`01-expert-coding-agent-workflows.md`](01-expert-coding-agent-workflows.md)：先规格、再实现、小步验证，是降低 review 成本的前置条件。
- [`../orchestration/01-when-to-use-subagents.md`](../orchestration/01-when-to-use-subagents.md)：AI review 是 subagent 的典型适用场景，因为它需要独立上下文和专项视角。
- [`../../playbooks/workflows/new-project-bootstrap.md`](../../playbooks/workflows/new-project-bootstrap.md)：TDD 是 AI review 可靠性的底座之一。
- [`../../playbooks/workflows/existing-project-onboarding.md`](../../playbooks/workflows/existing-project-onboarding.md)：既有项目需要先补上下文、测试和规则，AI reviewer 才不会乱猜。

可以把整体工作流理解成：

```text
Spec -> Plan -> Implement -> Self-review -> Tests -> Independent AI review -> Human risk review -> Merge
```

AI review 不是单独一环变魔法，而是整个 agent harness 的一个反馈回路。

## 最小可行版本

如果只想今天就开始，不用搭建 bot，按这个做：

1. 每个 agent 生成的改动完成后，先让它按“作者侧自审模板”审自己。
2. 新开一个会话，按“独立 AI reviewer 模板”审 diff。
3. 要求所有 findings 都有 `path:line`、失败场景和验证方式。
4. 只处理 `Critical / High / Medium`，忽略泛泛建议。
5. 人类最后看核心 diff、测试证据和 AI findings。
6. 每次把有用的 AI review 规则沉淀回 `AGENTS.md`。

这已经能解决很大一部分“AI 代码生成太快，人类 review 跟不上”的问题。

## 最终判断框架

当你问“这段代码能不能让 AI review”时，可以用下面这个判断：

| 问题 | 如果答案是“是” | 如果答案是“否” |
|---|---|---|
| diff 是否足够小？ | 可以 AI review | 先拆分 |
| 目标和非目标是否清楚？ | 可以审符合性 | 先补规格 |
| 是否有测试或可验证命令？ | AI findings 可被验证 | 先补验证 |
| 是否涉及高风险领域？ | AI 只做辅助 | 普通流程即可 |
| AI 评论是否证据化？ | 进入人类 triage | 要求重写 |
| 人类是否有时间看关键路径？ | 可以合并判断 | 不应合并 |

一句话收束：

```text
更快的 review 不是少看代码，
而是让人类只把注意力花在真正需要人类判断的地方。
```
