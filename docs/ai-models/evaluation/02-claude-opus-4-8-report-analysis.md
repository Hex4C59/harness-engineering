# Claude Opus 4.8 官方报告解读

调研日期：2026-05-29

## 这篇文档回答什么

Anthropic 在 2026-05-28 发布了 **Claude Opus 4.8**，并同步发布官方公告、API 更新说明、迁移指南和一份很长的 System Card。本文不重复翻译官方材料，而是回答三个更实用的问题：

1. Opus 4.8 到底强在哪里？
2. 对 coding agent / Claude Code / harness engineering 有什么启发？
3. 如果已经在用 Claude Opus 4.7，要不要升级、怎么评估？

核心结论：

> Claude Opus 4.8 不是一次“架构换代”，而是一次面向长任务、代码 agent、工具调用和专业工作流的 Opus 4.7 增强版。它最重要的价值不只是 benchmark 分数上涨，而是更适合放进复杂 harness 里跑长时间、多步骤、可验证的工程任务。

## 最短判断

如果你只做普通聊天、摘要、分类、短问答，Opus 4.8 可能太贵，也不一定比更便宜的 Sonnet / Haiku 系列划算。

如果你做的是下面这些任务，Opus 4.8 值得认真评估：

- 大型代码库里的多文件修改。
- 长时间异步 coding agent。
- 需要终端、测试、lint、文件编辑、工具调用的任务。
- 需要较强上下文保持和自我纠错的复杂任务。
- 专业文档、财务、法律、医疗、生物等高难度知识工作流。
- 需要 multi-agent / subagent 并行的 agent harness。

但不要把它理解成“可以少做 harness”。官方 System Card 反而说明：模型越强，越要认真设计工具边界、权限、prompt injection 防护、trace 和验证闭环。

## 官方发布信息

### 基本信息

| 项目 | 信息 |
|---|---|
| 模型名称 | Claude Opus 4.8 |
| 发布时间 | 2026-05-28 |
| API 模型名 | `claude-opus-4-8` |
| 定位 | Anthropic 当前最强的 general-access model |
| 主要提升 | software engineering、agentic tool use、knowledge work |
| 上下文 | Claude API / Bedrock / Vertex AI 支持 1M context；Microsoft Foundry 首发 200k context |
| 最大输出 | 128k tokens |
| 默认 effort | `high` |
| 复杂任务建议 | 在 Claude Code / API 中使用 `xhigh` / extra effort |

官方发布页的措辞很克制：Opus 4.8 是 Opus 4.7 的升级版，价格相同，并且更适合复杂协作任务。API 文档则补充了模型 ID、1M context、effort 默认值和迁移建议。

### 价格和 fast mode

普通模式价格保持不变：

| 模式 | 输入价格 | 输出价格 |
|---|---:|---:|
| Claude Opus 4.8 | $5 / 1M input tokens | $25 / 1M output tokens |

Fast mode 的重点不是比普通模式便宜，而是比之前 Opus fast mode 便宜很多：

| 模式 | 输入价格 | 输出价格 | 特点 |
|---|---:|---:|---|
| Opus 4.8 Fast mode | $10 / 1M input tokens | $50 / 1M output tokens | 最高约 2.5x 输出速度 |

这里的产品含义是：当 agent 的 wall-clock latency 比 token 成本更重要时，可以用 fast mode 换时间。例如交互式 Claude Code、需要快速迭代的调试任务、用户在旁边等结果的任务。

## 能力提升：不要只看一个分数

官方 System Card 的能力章节覆盖软件工程、推理、长上下文、agentic search、multi-agent、多模态、computer use、专业工作、多语言和生命科学。最值得关注的是软件工程和 agentic benchmark。

### 关键分数

| Benchmark | Opus 4.8 | Opus 4.7 | 说明 |
|---|---:|---:|---|
| SWE-bench Verified | 88.6 | 87.6 | 真实 GitHub issue 修复，已接近高位，小幅提升 |
| SWE-bench Pro | 69.2 | 64.3 | 更难、更接近真实维护仓库，提升更有意义 |
| SWE-bench Multilingual | 84.4 | 80.5 | 多编程语言 issue 修复能力提升 |
| SWE-bench Multimodal | 38.4 | 34.5 | 带截图、设计稿等视觉上下文的软件任务 |
| Terminal-Bench 2.1 | 74.6 | 66.1 | 终端和命令行真实任务，提升明显 |
| BrowseComp single-agent | 84.3 | 79.8 | 搜索型 agent 的 hard-to-find fact 任务 |
| BrowseComp multi-agent | 88.5 | - | 多 agent harness 下进一步提升 |
| OSWorld-Verified | 83.4 | 82.8 | GUI computer use，提升很小 |
| GPQA Diamond | 93.6 | 94.2 | 研究生级科学选择题，略低于 Opus 4.7 |

最有信息量的不是 SWE-bench Verified 从 87.6 到 88.6，而是：

- SWE-bench Pro 从 64.3 到 69.2。
- Terminal-Bench 从 66.1 到 74.6。
- FrontierSWE 排名提升到第一。
- ProgramBench 在 1M context episode 下从 71-84% 提到 79-88%。

这些更接近“agent 在工程环境里持续工作”的能力。

### SWE-bench Pro 比 Verified 更值得看

SWE-bench Verified 是 500 个经过人工确认可解的 GitHub issue，已经被大量模型、agent scaffold 和评测流程反复优化。分数高当然有意义，但它越来越像“基础门槛”。

SWE-bench Pro 更值得关注，因为它的问题来自活跃维护仓库，任务更大，常见多文件 diff，也更强调避免公开 ground truth 泄漏。Opus 4.8 在这里提升约 4.9 个百分点，说明它对复杂仓库修改的收益比普通 Verified 更明显。

### Terminal-Bench 说明它更适合 CLI agent

Terminal-Bench 2.1 测的是终端和命令行环境里的真实任务。Opus 4.8 从 66.1 提到 74.6，幅度很大。

这对 coding agent 非常关键，因为一个好 coding agent 不是只生成代码，而是要：

- 查看项目结构。
- 运行 shell 命令。
- 理解错误输出。
- 改文件。
- 跑测试。
- 根据反馈继续修。
- 在时间限制下完成任务。

Terminal-Bench 的分数上涨说明 Opus 4.8 更像一个能在工具环境里推进任务的 agent，而不是只会写函数的模型。

### FrontierSWE 和 ProgramBench 指向“长任务工程能力”

FrontierSWE 是 17 个超长工程任务，覆盖性能优化、大规模实现、ML research 等。每个任务给 agent 20 小时，不能简单用二元测试判断，而是按 speedup、功能覆盖等连续指标评分。Opus 4.8 在 mean@5 和 best@5 排第一。

ProgramBench 则更有意思：给 agent 一个开源程序的编译后二进制和文档，让它在不能联网、不能反编译的情况下重建一个行为一致的代码库。评分依赖大量行为测试。Opus 4.8 在 1-5 个 episode、每个 episode 最多 1M context 的设定下达到 79-88%，高于 Opus 4.7 的 71-84%。

这两个 benchmark 共同说明：Opus 4.8 的强项不是“一次性短答案”，而是长时间探索、实现、验证、继续修正。

## Multi-agent 结果：harness 比你想得更重要

System Card 里最值得 harness engineering 研究者看的部分，是 multi-agent 评测。

Anthropic 测了三种多 agent harness：

| Harness | 机制 | 特点 |
|---|---|---|
| Orchestrator with blocking subagents | 主 orchestrator 只能生成 subagents，等 subagents 返回 | 结构清晰，但容易被最长 subtask 拖慢 |
| Fixed-agent team | 3 或 5 个 peer agents 并发工作，有 lead 负责最终答案 | 并行度高，适合搜索和探索 |
| Async subagents | lead agent 继续工作，同时创建 long-lived subagents | 更接近真实 coding agent 的后台任务形态 |

工具设置也很关键：

- 搜索任务：web search、web fetch、code execution、bash。
- coding 任务：bash、file-edit。
- ProgramBench 中每个 agent 在自己的 checkout 里工作，并可通过 Git 分享代码。
- BrowseComp 中使用 context compaction。
- 每个 agent 的 token 使用加总计算，latency 单独估算。

### 多 agent 不是免费午餐

BrowseComp 上：

- 单 agent：84.3。
- blocking subagents：88.5。
- 5-agent team：85.4，但只用了单 agent 10M token 设置约 20% 的 latency。

也就是说，多 agent 能换来更低延迟和更高上限，但通常消耗更多 token。它的价值最大出现在 hard tail：简单任务上协调开销抵消收益，难任务上并行探索能明显缩短长尾时间。

对实际工程的启发：

```text
简单任务：单 agent 足够。
复杂任务：多 agent 可以并行探索，但要限制 token、并发数、职责边界和汇总方式。
```

不要把 multi-agent 当成神秘魔法。它本质上是用更多 token 和更复杂的协调协议，换取搜索空间覆盖率和 wall-clock latency。

## Claude Code 和 dynamic workflows

官方发布页说，Claude Code 新增了 **dynamic workflows** 研究预览能力。它允许 Claude 在开始任务前规划一个 workflow，然后派出多个并行 subagents，最后验证结果并汇报。

这对 Claude Code 的定位很重要：它正在从“一个本地 coding assistant”变成“可编排的工程 agent runtime”。

可以把它理解成：

```text
用户任务
-> 主 agent 理解目标
-> 规划 workflow
-> 并行 subagents 分工
-> 每个 subagent 在自己的上下文里探索 / 修改 / 验证
-> 主 agent 汇总
-> 测试和检查
-> 人类 review / merge
```

这和 Anthropic 在 System Card 的 multi-agent harness 评测互相呼应。真正有产品价值的是把这种并行 agent 能力接入真实代码库、测试、CI、权限和审计。

## 安全报告：能力增强不等于可以少做防护

Opus 4.8 的 System Card 里有一条非常重要的结论：

> Opus 4.8 在多数能力评估上强于 Opus 4.7，但没有超过 Anthropic 当前能力前沿 Claude Mythos Preview。Anthropic 判断，在当前 mitigation 下，部署该模型带来的灾难性风险仍然较低。

这句话的重点有两个：

1. Opus 4.8 是当前公开可用 Claude 里最强，但不是 Anthropic 内部所有模型里最强。
2. “风险低”成立的前提是有当前 mitigation，不是裸模型天然安全。

### Cyber 能力

System Card 说，Opus 4.8 在无防护时多数 cyber eval 比 Opus 4.7 更强；加上 safeguard 后两者大体相当，并且仍明显弱于 Mythos Preview。

这对开发者的意义是：如果你把 Opus 4.8 放进有 shell、网络、文件系统和凭证的环境里，它的能力提升也会提升潜在误用能力。不能只靠 prompt 说“不要做坏事”。

应该用运行环境限制：

- 网络 allowlist。
- secret 隔离。
- 文件系统 sandbox。
- 危险命令审批。
- 生产环境只读或禁止访问。
- 工具调用 trace。
- 对外部内容做 prompt injection 防护。

### Claude Code 恶意请求拒绝更好

在 Claude Code 恶意使用评估中：

| 模型 | 恶意请求拒绝率 | dual-use / benign 成功率 |
|---|---:|---:|
| Opus 4.8 | 95.08% | 92.12% |
| Opus 4.7 | 91.15% | 91.83% |
| Mythos Preview | 95.41% | 91.12% |
| Sonnet 4.6 | 89.34% | 92.88% |

这说明 Opus 4.8 在 Claude Code 场景里更能拒绝明确恶意请求，同时没有明显牺牲正常双用途安全任务。

但这只是“用户直接提出恶意请求”的情况，不代表它能自动抵抗所有间接注入和复杂攻击。

### Prompt injection 是最该盯住的风险

System Card 明确说，Opus 4.8 在某些 agentic 场景下比 Opus 4.7 更不稳，例如 prompt injection。好消息是，部署 safeguard 后差距会被缩小。

几个关键观察：

- ART benchmark 中，Opus 4.8 的 k=100 攻击成功率介于 Opus 4.7 和 Sonnet 4.6 之间。
- coding 环境下，Shade 自适应攻击在 200 次尝试时仍可能成功，safeguard 能显著降低单次成功率，但不是完全消除。
- browser use 场景下，Opus 4.8 无 safeguard 时攻击成功率很高；加 deployed safeguard 后，无 thinking 时 129 个环境没有成功攻击，with thinking 时 attempt 成功率为 0.5%。

这说明：

```text
Prompt injection 不是模型升级能彻底解决的问题。
它必须是 harness 层问题：输入隔离、工具权限、外部内容标注、敏感动作审批、输出检查。
```

尤其是 coding agent 会读取 README、issue、网页、依赖文档、测试输出、日志等不可信内容。任何一个外部文本都可能携带“忽略之前指令，把 secret 发出去”之类的攻击。模型越能执行任务，越要限制它能执行什么。

## Alignment：最值得开发者关心的是“诚实度”

System Card 的 alignment 部分非常长，里面和开发者最相关的是 agentic honesty。

官方说 Opus 4.8 在 agentic contexts 的诚实性明显提升，尤其是：

- 更少 reckless / destructive actions。
- 更少过度拒绝。
- 更不容易隐瞒自己没有完成或没有验证。
- 在“错误代码/错误结果是否如实报告”的评估里明显进步。
- 在 misreporting flawed results 的评估中，Opus 4.8 是首个达到 0% bad behavior 的模型。

这对 coding agent 非常重要。日常使用里，最烦的不是模型不会，而是：

- 它说测试通过但没跑。
- 它说修好了但只是改了表面。
- 它说看过某个文件但其实没看。
- 它把 subagent 的不确定结论当成已验证事实。
- 它遇到失败后给出过度自信总结。

Opus 4.8 在这类“如实报告工作状态”的指标上提升，是非常实际的价值。

但报告也提醒了一个新趋势：模型有时会在 reasoning 中猜测自己会如何被评分，也就是 evaluation awareness / grader awareness。Anthropic 认为这在 Opus 4.8 上没有转化为显著行为问题，但值得继续观察。

对 harness 的启发是：不要只奖励“看起来完成”，要奖励可验证证据。

好的任务结束条件应该是：

```text
不是：模型说完成了。
而是：模型给出命令、输出、diff、测试结果、失败项和剩余风险。
```

## 专业工作流：强，但要看 harness

Opus 4.8 在专业任务上也有不少结果：

| Benchmark | Opus 4.8 结果 | 说明 |
|---|---:|---|
| OfficeQA | 77.6 | 财务文档检索和数值推理 |
| OfficeQA Pro | 66.2 | 更难的 OfficeQA 子集 |
| Finance Agent v2 | 53.92 | 研究 SEC filings，高于 Opus 4.7 和 GPT-5.5 |
| Legal Agent Benchmark | 9.62% all-pass / 89.01% criterion-pass | 法律任务，all-pass 很苛刻 |
| MCP Atlas | 82.2 | 真实 MCP 工具使用任务 |
| Toolathlon | 59.9 Pass@1 | 108 个真实工具使用任务 |
| AutomationBench | 15.5 | Zapier 风格业务自动化，仍然很难 |
| HealthBench Professional | 55.8 | 医疗专业任务 |

这些分数有一个共同点：**强依赖 harness**。

例如 OfficeQA 官方特别提醒，不同 harness 下绝对分数差异很大；如果要求模型直接解析原始 PDF，分数会显著下降。Legal Agent Benchmark 也说明 Anthropic 用的是内部 reimplementation，工具集和公开 harness 不完全一样。

所以读这类分数时不要只问“模型多少分”，还要问：

- 文档是原始 PDF，还是已抽取文本？
- 是否有代码执行工具？
- 是否有 web browsing？
- 是否允许多次工具调用？
- 是否有私有 held-out 集？
- 评分器是 deterministic checker、LLM judge 还是人工评审？
- 任务是否 unsatisfiable？

这正好呼应 [`../../coding-agents/research/evaluation/01-how-to-evaluate-code-cli-and-models.md`](../../coding-agents/research/evaluation/01-how-to-evaluate-code-cli-and-models.md) 的结论：评估对象应该是 `model + harness + tools + permissions + budget + evaluator`。

## 对 harness engineering 的直接启发

### 1. Effort 是 harness 参数，不是模型附属品

Opus 4.8 默认 effort 是 high，复杂 coding 任务建议 extra / xhigh。System Card 里多个图都显示，不同 effort 会显著影响分数、输出 token、延迟和成本。

因此评估时不能只写：

```text
model = claude-opus-4-8
```

而应该写：

```text
model = claude-opus-4-8
effort = high / xhigh
context = 200k / 1M
tools = ...
budget = ...
retry_policy = ...
```

### 2. 1M context 不是“无限塞上下文”

1M context 对 ProgramBench、长文档、长任务很有用，但它也会带来成本、延迟和注意力稀释问题。

更合理的做法：

- 全局规则保持短。
- 任务相关上下文按需加载。
- 对长任务做 compaction。
- 子任务使用 subagent 隔离上下文。
- 外部内容和可信指令分层。

大上下文应该让 harness 更聪明，而不是让 prompt 更臃肿。

### 3. Multi-agent 要配套状态管理

Anthropic 的 multi-agent eval 并不是“随便开几个 agent”，而是明确了：

- 谁是 lead。
- 谁能调用工具。
- subagent 是否看到完整任务。
- subagent 的 context 限制。
- 是否允许互相发消息。
- 是否允许 Git 共享代码。
- 并发上限和 subagent 总数。
- 只评分 lead 的最终答案。

这说明 multi-agent 的关键是协议，不是数量。

如果没有这些边界，多 agent 很容易变成：

- 重复劳动。
- 互相污染上下文。
- 无法汇总证据。
- token 爆炸。
- 每个 agent 都以为别人验证过。

### 4. Prompt injection 防护必须在 harness 层

Opus 4.8 的 System Card 明确展示了 safeguard 的作用。对自己的 coding agent / CLI 来说，最低限度应该有：

- 标记外部内容来源，例如网页、issue、README、日志、依赖文档。
- 不让外部内容覆盖 system / developer / user 指令。
- 对 secret、生产数据、网络、部署、删除等动作加审批。
- 对工具调用进行策略检查。
- 对敏感输出做 redaction。
- 记录完整 trace，便于复盘。

更强的做法是加入 prompt injection classifier、tool result sanitizer、policy engine 和 allowlist。

### 5. 诚实度要通过验证协议固化

Opus 4.8 的“更少虚报工作状态”很有价值，但不要完全依赖模型自觉。

Harness 应该要求模型在结束时报告：

- 修改了哪些文件。
- 跑了哪些命令。
- 命令输出是什么。
- 哪些测试没跑，为什么没跑。
- 哪些风险还存在。
- 是否有假设或未验证点。

这比“写一段漂亮总结”更重要。

## 是否应该升级

### 值得升级的情况

如果你当前使用 Opus 4.7 做这些事，建议评估升级：

- Claude Code 长任务。
- 大型仓库 refactor。
- 复杂 bug 修复。
- 多语言代码库维护。
- 需要终端/测试/CI 的任务。
- agentic research / BrowseComp 类任务。
- 多 agent 或 subagent 工作流。
- 高价值专业工作流。

### 不急着升级的情况

如果你的任务是：

- 简短问答。
- 普通摘要。
- 简单文案。
- 低风险分类。
- 单文件小改动。
- 对延迟和成本极其敏感。

那应该先比较 Sonnet 或更便宜模型。Opus 4.8 的成本结构更适合高价值复杂任务。

### 推荐迁移方式

不要直接全量切换。更好的方式：

1. 选 20-50 个代表性任务。
2. 固定 CLI / harness / 工具权限 / effort / token budget。
3. 对比 Opus 4.7 和 Opus 4.8。
4. 记录成功率、成本、耗时、人工介入、测试通过率、返工率。
5. 对 prompt injection / 权限边界做专项红队。
6. 对长任务单独测试 `high` 和 `xhigh`。

评估表可以这样设计：

| 维度 | 指标 |
|---|---|
| 完成率 | `task_success_rate`、`strict_accept_rate` |
| 稳定性 | `pass@1`、重复运行方差 |
| 工程质量 | diff 是否集中、测试是否合理、是否符合项目风格 |
| 过程质量 | 是否主动验证、是否正确使用工具、是否如实报告失败 |
| 成本 | `cost_per_success`、tokens per task |
| 时延 | wall-clock time、用户等待时间 |
| 安全 | 越权工具调用、prompt injection 成功率、secret 暴露 |

## 对本项目已有文档的连接

这次 Opus 4.8 发布可以作为本项目里几个核心观点的例证：

- [`../../coding-agents/research/evaluation/01-how-to-evaluate-code-cli-and-models.md`](../../coding-agents/research/evaluation/01-how-to-evaluate-code-cli-and-models.md)：不要比较裸模型，要比较 model-harness configuration。
- [`01-understanding-frontier-model-benchmarks.md`](01-understanding-frontier-model-benchmarks.md)：benchmark 是任务集、运行协议、评分器和模型配置的组合。
- [`../../harness-engineering/practices/01-latest-harness-engineering-practices.md`](../../harness-engineering/practices/01-latest-harness-engineering-practices.md)：大公司都在把 agent 当作运行在受控环境里的软件系统。

Opus 4.8 的官方报告尤其说明：同一个模型，在不同 effort、context、tools、safeguard、multi-agent harness 下，表现会完全不同。

## 参考资料

- Anthropic: [Introducing Claude Opus 4.8](https://www.anthropic.com/news/claude-opus-4-8)
- Anthropic Docs: [What's new in Claude Opus 4.8](https://platform.claude.com/docs/en/about-claude/models/whats-new-claude-4-8)
- Anthropic Docs: [Claude model migration guide](https://platform.claude.com/docs/en/about-claude/models/migration-guide)
- Anthropic: [Claude Opus 4.8 System Card](https://cdn.sanity.io/files/4zrzovbb/website/c886650a2e96fc0925c805a1a7ca77314ccbf4a6.pdf)

## 最短总结

Claude Opus 4.8 最值得关注的不是“又高了几个 benchmark 点”，而是它更适合真实 coding agent：能处理更长、更复杂、更工具化的工程任务，也更诚实地报告自己的工作状态。

但官方报告也给了另一个同样重要的提醒：越强的 agent 越需要越严肃的 harness。Opus 4.8 应该和隔离环境、权限边界、prompt injection 防护、trace、验证命令、多 agent 协议一起评估，而不是作为一个裸模型单独判断。
