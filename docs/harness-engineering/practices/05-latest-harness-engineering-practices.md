# 最新 Harness Engineering 实践调研

调研日期：2026-05-29

## 这篇文档回答什么

这篇不是继续解释“harness engineering 是什么”，而是调研一批最新的工程博客、官方文档、技术报告和论文，看看真正把 agent 投入软件工程和生产工作流的团队，具体在做什么。

优先级如下：

1. 大公司或高质量团队的官方工程博客、官方文档、系统卡和技术报告。
2. 开源 agent / coding agent 项目的论文或架构报告。
3. 2026 年以后直接讨论 harness、agent runtime、agent observability、coding agent 的论文。
4. 社区博客只用来发现线索，不作为主要依据。

核心结论：

> 最新实践已经不再把 agent 当成“一个模型 + 一个 prompt”，而是把它当成运行在受控开发环境里的软件系统。真正的差异来自上下文治理、工具边界、执行环境、验证闭环、可观测 trace、权限策略和人类交接。

## 最重要的共同趋势

### 1. Harness 已经从 prompt 变成产品基础设施

OpenAI、Anthropic、GitHub、Google、Stripe、Microsoft 和 LangChain 的材料都在指向同一个方向：

```text
Agent = Model + Harness + Environment + Feedback
```

其中 harness 不只是 agent loop，而是：

- 线程 / run 生命周期。
- 工具调用协议。
- 权限和审批。
- 工作区隔离。
- 上下文选择。
- 任务状态。
- trace 和日志。
- 验证和评分。
- 人类 review / handoff。
- 失败样本回流 eval。

OpenAI 的 Codex App Server 文章尤其明确：同一个 Codex harness 要服务 web app、CLI、IDE extension、macOS app 和外部嵌入场景，因此 harness 被抽象成可复用的协议层，而不是藏在某一个 UI 里的实现细节。

### 2. 大公司都在把 agent 放进真实开发环境，而不是玩具 sandbox

Stripe Minions 运行在 Stripe 人类工程师已经使用的 devbox 上。GitHub Copilot coding agent 运行在 GitHub Actions 背景环境里。Google Jules 克隆代码到 Cloud VM。OpenAI Codex 和 Claude Code 都强调从仓库、终端、IDE、PR、CI 里拿真实信号。

这些系统的共同点是：

- agent 不是只看一段代码。
- agent 能读完整仓库上下文。
- agent 能运行项目命令。
- agent 能拿到测试、lint、CI、issue、PR、文档和内部工具反馈。
- agent 的最终产物通常是 patch、branch、draft PR 或 artifact，而不是聊天答案。

### 3. 真正可靠的系统混合了 deterministic workflow 和 agent loop

Stripe 的 Minions 把这个说得最清楚：它们的 blueprint 是状态机，里面有两类节点：

- deterministic code nodes：跑 lint、跑测试、push branch、处理 git。
- agent nodes：实现任务、修 CI 失败、理解上下文。

这个模式很关键：

```text
能用确定性代码做的，不要交给模型自由发挥。
只有无法预先写死的部分，才交给 agent loop。
```

这和 Anthropic 的 Building Effective Agents 一致：优先使用简单、可组合的 workflow，只有当任务真的需要模型自主决策时才使用 agent。

### 4. 可观测 trace 成为 harness 的核心资产

LangChain / LangSmith 的材料反复强调：agent observability 不是传统 request-response logging，因为 agent 失败往往发生在第 5、15、50 个工具调用之后。

需要记录：

- 用户输入。
- 上下文选择。
- 每次模型调用。
- 每次工具调用。
- 工具输入输出。
- 文件 diff。
- 命令输出。
- token、成本、耗时。
- 错误和重试。
- 人类审批。
- 最终 artifact。

LangChain 的一句话很有代表性：

> In software, the code documents the app; in AI, the traces do.

这和 2026 年的 Harness-Bench、AI Harness Engineering、Agentic Harness Engineering 论文完全同向：trace 不只是 debug 日志，而是评估、归因、回放、训练和 harness 演化的基础数据结构。

### 5. 权限边界从“是否允许”变成“如何分层允许”

最新实践不是简单地“agent 能不能跑命令”，而是按风险分层：

- 在隔离 devbox / cloud VM 里可以给更大权限。
- 对生产数据、secret、外网、部署、删除、支付、邮件等动作必须收紧。
- GitHub Copilot coding agent 使用受限开发环境、firewall、GitHub Actions 限制和隐藏字符过滤。
- Stripe Minions 运行在 QA/devbox 环境，没有生产数据和任意网络出口。
- Claude Code 引入 hooks、permissions、auto mode 等机制，把审批做成可配置层。

实践结论：

```text
不要只靠 prompt 说“不要做危险事”。
要让运行环境本身限制危险事。
```

### 6. 上下文治理从“塞更多”变成“按作用域加载”

Claude Code best practices 强调 context window 会填满，填满后性能下降。Stripe 也明确避免把全局规则都塞给 agent，而是使用按目录或文件 pattern 触发的 rule files。

常见做法：

- `AGENTS.md` / `CLAUDE.md` 只放入口地图和全局规则。
- 细节放到更局部的文档、skills、commands、rules。
- 按目录、文件类型、任务类型加载规则。
- 子任务用 subagent 或 scoped prompt 隔离上下文。
- 旧对话要 compaction，而不是无限追加。

这和本项目现在的约定完全一致：`AGENTS.md` 不应该变成百科全书。

### 7. “模型评估”正在变成“model-harness configuration 评估”

Harness-Bench 直接提出：agent 能力应该报告在 `model + harness` 配置级别，而不是只归因给 base model。

这点在实践中也很明显：

- SWE-bench 分数受 agent scaffold、工具、重试、测试策略影响。
- Terminal-Bench 分数受 CLI/harness 和资源隔离影响。
- Stripe Minions 的成功来自 devbox、blueprint、Toolshed、rules、CI，不是模型单独完成。
- OpenAI Codex App Server 的价值是让多个 surface 共享同一个 harness。

所以最新实践的评估对象越来越像：

```text
model + prompt + tools + permissions + workspace + lifecycle + evaluator + retry policy
```

## 来源地图

| 来源 | 类型 | 主要内容 | 对 harness 的启发 |
|---|---|---|---|
| OpenAI: Harness engineering | 工程博客 | Codex 时代软件团队如何设计环境、意图和反馈循环 | harness 是 agent-first 软件工程的核心工作 |
| OpenAI: Codex App Server | 工程博客 | Codex harness 如何跨 web、CLI、IDE、macOS 和 SDK 复用 | agent loop、thread lifecycle、typed items、approval、tool execution 要协议化 |
| Anthropic: Building Effective Agents | 工程博客 | workflow 与 agent 的边界、工具设计、eval、复杂度控制 | 先 workflow，后 agent；工具文档和测试是 ACI 的一部分 |
| Anthropic: Claude Code best practices | 工程博客 / docs | CLAUDE.md、上下文管理、TDD、并行、slash commands、subagents、hooks | context window 是稀缺资源；把工作流沉淀成项目文件 |
| GitHub Copilot coding agent | 官方文档 / 产品博客 | issue 到 draft PR 的异步 coding agent | 背景 agent 需要 sandbox、firewall、Actions 限制和 human review |
| Google Gemini CLI / Jules / Antigravity | 官方博客 / docs | terminal agent、GitHub Actions、Cloud VM、server-side harness | terminal 是入口，长期方向是多 surface、server-side harness 和团队协作 |
| Stripe Minions | 工程博客 | 大规模 unattended coding agents，devbox、blueprint、Toolshed、CI loop | 最接近生产级 harness 实践案例 |
| Microsoft Magentic-One / Magentic-UI | 研究报告 / 博客 | Orchestrator + specialist agents、人类监督、web/file/code tools | 多 agent 需要进度跟踪、replan、人类可理解状态 |
| Microsoft Agent Lightning | 研究博客 / repo | 解耦 agent execution 和 RL training，trace spans 作为训练数据 | agent trace 可以成为训练接口 |
| LangChain / LangSmith / LangGraph | 工程博客 / docs | 生产 agent observability、durable execution、HITL、eval loop | trace -> dataset -> eval -> redeploy 是生产闭环 |
| SWE-agent / OpenHands / Terminal-Bench | 论文 / 开源项目 | agent-computer interface、sandbox、真实任务评估 | ACI 和执行环境是软件工程 agent 的真实边界 |
| Harness-Bench / AI Harness Engineering / AHE / Meta-Harness | 论文 | harness 作为研究对象、trace-based eval、自动 harness 演化 | harness 本身可以被评估、优化和自动改进 |

## OpenAI：Codex 里的 harness engineering

### 他们解决的问题

OpenAI 的 Codex 不是一个单一 CLI，而是一套能运行在多种产品表面的 software engineering agent：

- ChatGPT web app。
- Codex CLI。
- IDE extension。
- macOS app。
- SDK / App Server。

当同一个 agent 要跨多个 surface 工作时，不能把 agent loop 写死在 UI 里。OpenAI 因此把 Codex 的 harness 抽象出来，使其能够被多个客户端复用。

### 关键实践

#### 1. Agent loop 协议化

Codex App Server 文章提到，Codex harness 需要表达多种 item：

- user message。
- agent message。
- tool execution。
- approval request。
- diff。
- lifecycle event。

每个 item 都有类型和生命周期。这个设计让客户端不需要猜 agent 在做什么，而是可以基于结构化协议渲染、审批、恢复和追踪。

实践启发：

```text
不要只把 agent 输出当成字符串流。
要把 agent run 拆成 typed events。
```

#### 2. 多 surface 共享同一个 harness

OpenAI 的 App Server 目标之一是让外部团队嵌入同一个 Codex harness，而不是为 web、CLI、IDE 各写一套 agent 实现。

这带来几个好处：

- 能复用工具执行逻辑。
- 能复用 approval 和 sandbox 策略。
- 能复用线程生命周期。
- 能复用 trace。
- 能让不同 UI 只关注展示和交互。

实践启发：

```text
把 agent runtime 做成平台能力，而不是某个聊天界面的附属实现。
```

#### 3. 仓库知识和工具是事实来源

OpenAI 的 harness engineering 文章强调，人类不应该一直复制粘贴上下文给 agent。Codex 应该直接使用标准开发工具，例如 `gh`、本地脚本、repo-embedded skills，自己获取上下文。

实践启发：

- 把项目事实写进仓库。
- 把常见操作写成脚本。
- 让 UI、日志、指标、CI 输出对 agent 可读。
- 用 lint、测试和结构化检查防止漂移。

### 可迁移模式

| OpenAI 模式 | 可以怎么学 |
|---|---|
| Typed event stream | 本地 agent 也可以记录 `message/tool/diff/approval/result` |
| Shared App Server | 多个入口共用同一个 agent runtime |
| Repo-embedded skills | 把项目规则、命令、检查写进仓库 |
| Tool-readable development environment | 让 agent 直接读 issue、PR、CI、日志，而不是靠人转述 |
| Feedback loop | 失败要进入 docs、tests、lint 或 eval |

## Anthropic：Claude Code 的 agentic coding 实践

### 他们解决的问题

Claude Code 是一个 terminal-first 的 agentic coding environment。Anthropic 的 best practices 和 docs 不是单纯讲 prompt，而是讲如何把 agent 放进真实工程工作流。

### 关键实践

#### 1. `CLAUDE.md` 是入口，不是百科全书

Anthropic 建议把项目常用命令、架构说明、风格规则、测试方式写入 `CLAUDE.md`，但也强调上下文窗口会填满，性能会下降。

实践启发：

```text
入口文件应该短、稳定、可导航。
细节应该分散到更局部、更任务化的文件。
```

这和本项目 `AGENTS.md` 的约定一致。

#### 2. 先规划，再执行

Claude Code 文档强调让 agent 先读代码、列计划、再修改。对于复杂任务，可以要求：

- 先探索。
- 不写代码。
- 输出计划。
- 人类确认。
- 再执行。

实践启发：

```text
把探索、计划、实现、验证拆成阶段。
不要让 agent 一上来就改文件。
```

#### 3. TDD 在 agentic coding 中更有价值

Anthropic 明确提到 test-driven development 对 agentic coding 特别有效：

1. 先让 agent 写失败测试。
2. 人类确认测试覆盖需求。
3. 再让 agent 实现。
4. 运行测试闭环。

这比“请实现功能”更可靠，因为 agent 有一个可执行目标。

#### 4. Slash commands、subagents、hooks、MCP 是 harness 配置层

Claude Code 的配置能力本质上是 harness engineering：

- slash commands：把重复工作流变成版本化 prompt。
- subagents：隔离上下文和工具权限。
- hooks：在工具调用前后插入确定性检查。
- MCP：接入外部工具和数据源。
- settings：管理权限、模型、环境。

实践启发：

```text
不要把所有行为塞进一个系统提示。
用 commands、hooks、subagents、tools 分解行为。
```

### 可迁移模式

| Claude Code 模式 | 可以怎么学 |
|---|---|
| `CLAUDE.md` | 本项目用 `AGENTS.md` 做入口地图 |
| Slash commands | 把常见调研、写文档、更新目录流程做成命令 |
| Hooks | 在保存前检查链接、路径、格式 |
| Subagents | 让调研、写作、审稿分离上下文 |
| TDD workflow | 先写验收标准，再让 agent 实现 |

## GitHub：Copilot coding agent 的异步 PR 工作流

### 他们解决的问题

GitHub Copilot coding agent 的核心场景是：

```text
把一个 GitHub issue 分配给 agent
-> agent 在背景环境里工作
-> agent 创建 draft pull request
-> 人类 review 和 merge
```

这不是聊天助手，而是 GitHub workflow 里的异步开发者。

### 关键实践

#### 1. PR 是 agent 的主要交付物

GitHub 把 agent 输出收敛到 draft PR，而不是“聊天里贴一段 patch”。这带来几个好处：

- 复用 GitHub review 流程。
- 复用 CI。
- 复用 branch protection。
- 复用 audit log。
- 人类在熟悉的地方审查。

实践启发：

```text
让 agent 产物进入已有工程流程，而不是发明一套孤立交付方式。
```

#### 2. Restricted development environment

GitHub 文档明确提到 Copilot coding agent 工作在 sandbox development environment 中，internet access 受 firewall 控制。

还包括：

- 限制谁能 assign task。
- 限制 GitHub Actions workflow runs。
- hidden characters filtering。
- blocked firewall request 会反馈到 PR/comment。

实践启发：

```text
agent 的安全不是靠“模型听话”，而是靠环境和平台策略。
```

#### 3. MCP 扩展能力，但也扩大风险面

GitHub 支持通过 MCP 扩展 coding agent，让它获取外部上下文。但这也意味着 tool surface 要治理：

- 哪些 MCP server 可以接入？
- 谁配置？
- tool 输出是否可信？
- 是否能访问 secret？
- 是否能执行 destructive action？

实践启发：

```text
MCP 是能力层，也是攻击面。
```

## Google：Gemini CLI、Jules、Antigravity 的路径

### 他们解决的问题

Google 的公开实践分成几层：

- Gemini CLI：把 Gemini 放进终端，作为本地 open-source agent。
- Gemini CLI GitHub Actions：把 CLI 放进 GitHub repo 的异步协作环境。
- Jules：异步 coding agent，克隆代码到 Cloud VM 并验证修改。
- Antigravity：Google 正在推动的 agent-first development platform，并强调 server-side harness。

### 关键实践

#### 1. CLI 是入口，不是终点

Gemini CLI 起点是 terminal agent，但 Google 后续把它扩展到 GitHub Actions、IDE、Antigravity。2026 年 5 月 Google 的更新提到把 Gemini CLI 迁移到 Antigravity CLI，并提到 server-side harness。

实践启发：

```text
terminal agent 很适合个人探索，但团队级 agent 需要服务端 harness、权限、协作和状态。
```

#### 2. 异步 agent 需要云端工作区

Jules 的公开页面强调：

- clone code in a Cloud VM。
- verify changes work。
- async multi-agent development。

这和 GitHub Copilot、OpenAI Codex cloud tasks、Stripe devboxes 是同一个方向：让 agent 在可复制、可隔离、可审计的远端环境里跑。

#### 3. GitHub Actions 成为 agent harness 的天然宿主

Gemini CLI GitHub Actions 用于：

- issue triage。
- PR review。
- on-demand collaboration。
- repo 内异步任务。

GitHub Actions 有天然优势：

- 事件触发。
- secret 管理。
- job 日志。
- 权限模型。
- PR 集成。

但相关安全论文也说明，GitHub event context 进入 agent prompt 后，会带来 agentic workflow injection 风险。

## Stripe：Minions 是目前最值得读的生产案例

### 他们解决的问题

Stripe Minions 是 homegrown unattended coding agents。Stripe 在工程博客中报告：Minions 每周产生超过 1,300 个 merged PR，这些 PR human-reviewed，但没有 human-written code。

这是目前公开资料里最接近“真实大公司 agentic coding at scale”的案例之一。

### 关键实践

#### 1. Devbox：给 agent 和人类同一套工程环境

Stripe 的 devbox 是 AWS EC2 实例，里面有：

- 源码。
- 服务。
- 缓存。
- Bazel / type checking / codegen 等预热状态。
- QA 环境。
- 隔离工作区。

Stripe 早就为人类工程师建设了 devboxes，后来发现这也是 agent 的理想执行环境。

实践启发：

```text
先把人类开发环境标准化、可复制、可快速启动。
再把 agent 放进去。
```

这很重要：如果人类开发环境都不可复制，agent 只会把混乱放大。

#### 2. Blueprint：workflow 和 agent 的混合体

Stripe Minions 的 blueprint 是一套状态机：

- deterministic nodes：lint、git、push、CI、autofix。
- agent nodes：implement task、fix CI failures。

实践启发：

```text
把确定性步骤写成代码。
把不确定性步骤交给 agent。
```

这样既能减少 token 和 CI 成本，也能降低 agent 犯错空间。

#### 3. Scoped rule files

Stripe 不把所有规则都全局塞给 agent，而是使用按目录或 file pattern 触发的 rule files。这样 agent 走到某个子目录时，才加载对应规则。

实践启发：

```text
规则越大，越需要按作用域加载。
```

对于超大仓库，这可能比“写一个巨大的 AGENTS.md”重要得多。

#### 4. Toolshed：集中 MCP 工具层，但按任务裁剪

Stripe 构建了内部 MCP server：Toolshed。它让不同 agent 共享内部工具，包括文档、ticket、build status、code intelligence 等。

但 Stripe 同时强调：

- 工具太多会让 agent 变差。
- 默认只给 Minions 小而相关的工具集。
- 用户可以按主题增加工具组。
- 安全控制框架阻止 destructive actions。

实践启发：

```text
工具平台可以集中建设。
每个 agent 的可见工具必须裁剪。
```

#### 5. Shift feedback left

Stripe 不让 agent 一直等 CI 反馈，而是尽量本地化、前置化：

- 本地 lint。
- autofix。
- pre-push hooks。
- 缓存 lint 结果。
- blueprint 中确定性执行部分 checks。
- 只有有限次数进入完整 CI loop。

实践启发：

```text
越早给 agent 可执行反馈，越便宜、越稳定。
```

### Stripe 案例的核心教训

| 问题 | Stripe 的答案 |
|---|---|
| agent 在哪里跑？ | 隔离、可复制、预热的 devbox |
| agent 怎么组织长任务？ | blueprint 状态机 |
| agent 怎么拿上下文？ | scoped rule files + Toolshed MCP |
| agent 怎么验证？ | local lint/autofix + limited CI loop |
| agent 能不能乱来？ | QA 环境、无生产数据、无任意网络出口、安全控制 |
| agent 产物是什么？ | human-reviewed PR |

## Microsoft：Magentic-One、Magentic-UI、Agent Lightning

### Magentic-One：Orchestrator + specialist agents

Magentic-One 是 Microsoft Research 的 generalist multi-agent system。它的架构是：

- Orchestrator：规划、分配任务、跟踪进度、失败后 re-plan。
- WebSurfer：浏览网页。
- FileSurfer：浏览文件。
- Coder：写代码。
- ComputerTerminal：执行终端命令。

实践启发：

```text
多 agent 的关键不是“多几个角色名”，而是要有一个能追踪任务状态并重新规划的 orchestrator。
```

### Magentic-UI：人类监督不是最后 approve 一下

Magentic-UI 强调 human-centered agentic systems。它关心：

- 用户是否知道 agent 正在做什么。
- 用户能不能理解 agent 的计划。
- 用户能不能中途干预。
- agent 失败时是否可恢复。

实践启发：

```text
Human-in-the-loop 不应该只是最终审批按钮。
它应该贯穿计划、执行、风险动作和恢复。
```

### Agent Lightning：trace 变成训练接口

Agent Lightning 的核心是解耦：

- agent execution。
- reinforcement learning training。

它通过 runner/tracer 收集 execution spans，再把这些 spans 转换为训练数据。它支持接入现有框架，而不是要求重写 agent。

实践启发：

```text
如果 trace 结构足够好，它不仅能 debug，还能训练和优化 agent。
```

这和 Harness-Bench、AHE、LangSmith 的方向一致：trace 是 agent 系统的核心资产。

## LangChain / LangSmith / LangGraph：生产 agent 的运行时要求

### 他们解决的问题

LangChain 的近年材料不再只讲 chain，而是在讲生产 agent 的基础设施：

- tracing。
- evaluation。
- production monitoring。
- durable execution。
- memory。
- scheduled runs。
- sandboxed code execution。
- human-in-the-loop。
- deployment runtime。

### 关键实践

#### 1. Trace 驱动的改进循环

LangSmith 的实践循环是：

```text
production trace
-> 找到失败样本
-> 加入 dataset / annotation queue
-> 修改 prompt / tool / harness
-> 跑 experiment
-> 对比新旧版本
-> redeploy
```

这就是 production agent 的 eval flywheel。

#### 2. Durable execution

LangGraph 强调 agent 运行不是一次 HTTP 请求，而是可暂停、可恢复、可重试的长过程。

需要：

- checkpoint。
- interrupt/resume。
- human approval。
- durable state。
- time travel / replay。

实践启发：

```text
长任务 agent 不能只靠内存里的 while loop。
```

#### 3. Runtime 和 harness 分工

LangChain 的 deep agents runtime 文章把 runtime 能力列得很直接：

- durable execution。
- memory。
- multi-tenancy。
- guardrails。
- human-in-the-loop。
- observability。
- sandboxed code execution。
- scheduled runs。

其中 harness 是围绕模型帮助 agent 在特定 domain 成功的系统；runtime 是承载这些能力的运行基础设施。

## Open-source coding agent：SWE-agent、OpenHands、Goose

### SWE-agent：Agent-Computer Interface

SWE-agent 的重要贡献是强调 **Agent-Computer Interface**。同一个模型，如果给它更适合 agent 使用的 shell/editor interface，SWE-bench 表现会明显变化。

实践启发：

```text
工具界面的形状会改变模型能力。
```

这和 harness engineering 的核心假设一致：不是只有模型重要，agent 看到什么、怎么操作也重要。

### OpenHands：开放的软件工程 agent 平台

OpenHands 把软件工程 agent 做成一个通用平台，强调：

- sandbox。
- command execution。
- browser。
- file editing。
- evaluation。
- extensibility。

它适合作为研究和工程对照：如果想看一个完整 software engineering agent 平台怎么组织工具和环境，OpenHands 是重要样本。

### Goose：MCP-native 的本地 agent

Goose 起源于 Block，后来被 Stripe Minions fork 作为基础。Goose 的重点是：

- open-source。
- desktop + CLI + API。
- MCP extension。
- 本地执行。
- 多模型支持。

实践启发：

```text
一个好的 agent harness 应该让工具接入成为一等公民。
```

## 2026 年论文线索：harness 正在成为研究对象

### AI Harness Engineering

这篇论文把 harness 定义为 foundation-model software agent 的 runtime substrate，并列出 11 个责任：

1. task specification。
2. context selection。
3. tool access。
4. project memory。
5. task state。
6. observability。
7. failure attribution。
8. verification。
9. permissions。
10. entropy auditing。
11. intervention recording。

它还提出 H0-H3 的成熟度阶梯，以及 trace-based episode package。

实践启发：

```text
每次 agent run 都应该能打包成可审计 episode。
```

### Harness-Bench

Harness-Bench 明确研究不同 harness configuration 对 agent 表现的影响。它构造了 106 个 sandboxed offline tasks，记录 final artifacts、execution traces、usage statistics 和 validator outputs。

它的核心结论是：

> Agent capability should be reported at the model-harness configuration level.

实践启发：

```text
比较 agent 时，不要只写模型名。
要写 harness、工具、预算、权限、环境和评估器。
```

### Agentic Harness Engineering

AHE 研究自动演化 coding-agent harness。它提出三个 observability pillars：

- component observability：每个可编辑 harness component 都有文件级表示，可 revert。
- experience observability：把原始轨迹压缩成 agent 能消费的 evidence corpus。
- decision observability：每次 harness edit 都要写出预测，再用下一轮结果验证。

实践启发：

```text
harness 也可以像代码一样被 agent 修改。
但前提是组件、经验和决策都可观察。
```

### Meta-Harness

Meta-Harness 把 harness optimization 做成外层搜索：让一个 agent 修改另一个 agent 的 harness，并用 benchmark 反馈优化。

实践启发：

```text
未来模型升级之外，还有 harness 升级。
同一个模型可能因为 harness 不同产生巨大差异。
```

### Terminal-Bench

Terminal-Bench 不是 harness 论文，但它很适合观察 harness 效应。它让不同 CLI agent 和模型在命令行任务中比较，任务涉及软件工程、数据、系统配置、机器学习等。

实践启发：

```text
terminal agent 的能力高度依赖 shell 工具、环境、超时、日志和恢复策略。
```

### Agentic workflow security papers

2026 年有多篇论文开始关注 GitHub Actions、issue、PR comment、MCP tools 中的 agentic workflow injection。

常见风险：

- issue body 里藏 prompt injection。
- PR comment 引导 agent 泄露 secret。
- tool 输出带恶意指令。
- agent 执行 out-of-scope action。
- workflow 把不可信上下文直接拼进 prompt。

实践启发：

```text
所有外部文本都是 untrusted input。
所有工具输出也应该当作 untrusted input。
```

## 最新实践里的设计模式

### Pattern 1：Agent 运行在隔离工作区

代表：

- Stripe devbox。
- GitHub Actions sandbox。
- Google Cloud VM。
- OpenAI Codex cloud task。
- OpenHands sandbox。

落地要点：

- 每个任务一个干净 workspace。
- 无生产数据。
- 默认无任意外网出口。
- 分支 / worktree 隔离。
- 可以重建。
- 可以保存 artifact 和 trace。

### Pattern 2：规则按作用域加载

代表：

- Stripe scoped rule files。
- Claude Code `CLAUDE.md` + commands + subagents。
- OpenAI repo-embedded skills。
- 本项目 `AGENTS.md`。

落地要点：

- 根规则短。
- 目录规则细。
- 任务规则可复用。
- 不重复规则。
- 规则版本化。

### Pattern 3：工具层集中建设、调用面裁剪

代表：

- Stripe Toolshed。
- GitHub MCP。
- Gemini CLI extensions。
- Claude Code MCP。
- Goose MCP。

落地要点：

- 工具注册统一。
- 工具说明短而准确。
- 工具名称唯一且语义清楚。
- 每个 agent 只暴露必要工具。
- destructive tools 需要权限门。
- 工具输出要有结构化格式。

### Pattern 4：确定性检查包围 agent loop

代表：

- Stripe blueprints。
- GitHub PR + CI。
- Claude Code hooks。
- LangGraph workflow nodes。

落地要点：

- lint、format、test、typecheck 不要让模型“记得做”，要由 harness 调用。
- git push、PR creation、artifact upload 最好做成确定性节点。
- agent 只处理无法预先写死的 reasoning/editing。

### Pattern 5：Trace 是默认产物

代表：

- LangSmith。
- Codex App Server typed items。
- Harness-Bench episode traces。
- Agent Lightning spans。
- AHE evidence corpus。

落地要点：

- 每次运行都保存 trace。
- trace 关联 final diff 和验证结果。
- 失败 trace 能一键进入 eval set。
- trace 可以被 agent 读取用于下一轮改进。

### Pattern 6：人类介入是中途机制，不只是最终审查

代表：

- Magentic-UI。
- Claude Code approvals/hooks。
- GitHub draft PR review。
- LangGraph interrupt/resume。

落地要点：

- 高风险工具调用前审批。
- 不确定需求时提问。
- 长任务中展示计划和当前状态。
- 最终 PR 仍由人类 review。

### Pattern 7：并行 agent 必须隔离

代表：

- Codex 多任务并行。
- Stripe 多 devbox。
- Claude Code parallel sessions with worktrees。

落地要点：

- 每个 agent 独立 branch/worktree/devbox。
- 不共享可变工作目录。
- 不共用敏感 token。
- 并行结果通过 PR 或 patch 合并。

### Pattern 8：失败要变成 harness 改进

代表：

- LangSmith trace-to-dataset。
- OpenAI failure feedback。
- AHE self-declared prediction verification。
- Stripe lint/CI feedback loop。

落地要点：

- 每次失败分类。
- 可重复失败写成 eval。
- 缺少上下文就补文档。
- 缺少检查就补测试/lint。
- 工具误用就改工具说明或权限。

## 反模式

### 反模式 1：一个巨大的全局说明文件

问题：

- 占满上下文。
- 规则互相冲突。
- agent 不知道哪些规则当前相关。
- 难维护。

更好的做法：

- 入口文件做地图。
- 细节按目录、任务、工具拆分。
- 用 commands / skills / subagents 表达 workflow。

### 反模式 2：把所有工具都给 agent

问题：

- 工具选择变难。
- token 开销变大。
- 安全面扩大。
- 错误工具调用更多。

更好的做法：

- 工具按任务裁剪。
- 默认最小权限。
- destructive tool 必须审批。
- tool outputs 结构化。

### 反模式 3：只看最终答案，不看轨迹

问题：

- 不知道失败原因。
- 无法复现。
- 无法改善 harness。
- agent 可能靠偶然过关。

更好的做法：

- 保存完整 trace。
- 记录工具调用和 diff。
- 把失败 trace 进入 eval。

### 反模式 4：把 CI 当作唯一反馈

问题：

- 太慢。
- 太贵。
- 失败反馈太晚。
- agent 在 CI 上盲目重试。

更好的做法：

- 本地 lint/typecheck/test 前置。
- 有 autofix 就 deterministic 运行。
- CI 只做最后或有限轮验证。

### 反模式 5：让 agent 在真实生产权限下试错

问题：

- 误删数据。
- 泄露 secret。
- 越权调用。
- 供应链风险。

更好的做法：

- dev/QA 环境。
- sandbox。
- firewall。
- no production data。
- no arbitrary egress。
- human approval for irreversible actions。

### 反模式 6：用 benchmark 名字掩盖配置差异

问题：

- “SWE-bench 70%”可能来自完全不同 harness。
- 模型、工具、重试、预算、环境都可能不同。

更好的做法：

```text
报告 model + harness + tools + budget + retry + environment + evaluator
```

## 对个人和小团队的落地建议

不需要一开始复制 Stripe 或 OpenAI。可以按成熟度升级。

### Level 1：仓库可读

- 根目录有 `AGENTS.md`。
- `README.md` 清楚。
- 常用命令写清楚。
- 文档按主题组织。
- 当前状态和计划可读。

### Level 2：验证可执行

- 有一键 test/lint/typecheck。
- agent 修改后必须运行验证。
- 文档链接可以检查。
- 失败有记录。

### Level 3：工作区隔离

- 每个任务用 git branch 或 worktree。
- 高风险任务用 container/devbox。
- 生成产物和缓存不混进主目录。

### Level 4：工具和规则可裁剪

- 规则按目录或主题拆。
- 常见 workflow 写成脚本或命令。
- 工具最小暴露。
- destructive actions 需要人工确认。

### Level 5：Trace 和 eval 闭环

- 保存 agent run 的关键 trace。
- 失败样本进入 eval。
- 每次改 harness 前后比较。
- 不靠感觉评价 agent。

## 对本项目的启发

本项目是资料和示例代码工作区，不是生产 agent runtime。但可以先把 harness practice 自己用起来：

- `AGENTS.md` 继续保持入口地图。
- 每个主题目录有自己的 `README.md`。
- 新调研文档写入参考资料，并标注调研日期。
- 遇到最新模型、榜单、产品能力时联网确认。
- 未来新增 `examples/` 时，每个示例必须有运行方式和验证方式。
- 如果做 agent eval 示例，记录 `model + CLI + tools + budget + task suite + scoring`。
- 如果做 coding agent 示例，至少包含 sandbox、trace、verification 和 permission 的最小实现。

## 一个可复用的 harness 设计清单

设计一个 coding agent harness 时，至少回答这些问题：

| 问题 | 推荐答案 |
|---|---|
| agent 在哪里跑？ | 隔离 workspace、container、devbox 或 CI runner |
| agent 能看到什么？ | README、AGENTS、局部规则、必要代码、任务说明 |
| agent 能调用什么？ | 最小工具集，按任务裁剪 |
| agent 怎么修改？ | patch / file edit / branch |
| agent 怎么验证？ | lint、test、typecheck、hidden eval |
| agent 怎么保存状态？ | thread/run state、plan、trace、artifact |
| agent 怎么失败？ | 失败分类、日志、可恢复状态 |
| 哪些操作要审批？ | 删除、部署、生产数据、secret、外网、支付、邮件 |
| 人类在哪里介入？ | plan review、risk action approval、PR review |
| 如何比较配置？ | 固定任务集，报告成本、成功率、trace、失败原因 |

## 最值得继续读的来源

### 官方工程博客和文档

- OpenAI: [Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/)
- OpenAI: [Unlocking the Codex harness: how we built the App Server](https://openai.com/index/unlocking-the-codex-harness/)
- OpenAI: [Introducing Codex](https://openai.com/index/introducing-codex/)
- OpenAI: [Introducing upgrades to Codex](https://openai.com/index/introducing-upgrades-to-codex/)
- Anthropic: [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- Anthropic: [Claude Code best practices](https://www.anthropic.com/engineering/claude-code-best-practices)
- Anthropic Docs: [Claude Code common workflows](https://docs.anthropic.com/en/docs/claude-code/common-workflows)
- Anthropic Docs: [Claude Code subagents](https://docs.anthropic.com/en/docs/claude-code/sub-agents)
- Anthropic Docs: [Claude Code hooks](https://docs.anthropic.com/en/docs/claude-code/hooks)
- GitHub Docs: [About GitHub Copilot coding agent](https://docs.github.com/en/copilot/using-github-copilot/coding-agent/about-assigning-tasks-to-copilot)
- GitHub Blog: [GitHub Copilot: meet the new coding agent](https://github.blog/news-insights/product-news/github-copilot-meet-the-new-coding-agent/)
- Google: [Gemini CLI: your open-source AI agent](https://blog.google/technology/developers/introducing-gemini-cli-open-source-ai-agent)
- Google: [Gemini CLI GitHub Actions](https://blog.google/technology/developers/introducing-gemini-cli-github-actions)
- Google: [Build with Jules, your asynchronous coding agent](https://blog.google/innovation-and-ai/models-and-research/google-labs/jules/)
- Google Developers: [Transitioning Gemini CLI to Antigravity CLI](https://developers.googleblog.com/an-important-update-transitioning-gemini-cli-to-antigravity-cli/)
- Stripe: [Minions: Stripe's one-shot, end-to-end coding agents](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents)
- Stripe: [Minions Part 2](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2)
- Microsoft Research: [Magentic-One](https://www.microsoft.com/en-us/research/publication/magentic-one-a-generalist-multi-agent-system-for-solving-complex-tasks/)
- Microsoft Research: [Magentic-UI](https://www.microsoft.com/en-us/research/blog/magentic-ui-an-experimental-human-centered-web-agent/)
- Microsoft Research: [Agent Lightning](https://www.microsoft.com/en-us/research/blog/agent-lightning-adding-reinforcement-learning-to-ai-agents-without-code-rewrites/)
- LangChain: [Agent observability in production](https://www.langchain.com/blog/production-monitoring)
- LangChain: [The agent improvement loop starts with a trace](https://www.langchain.com/blog/traces-start-agent-improvement-loop)
- LangChain: [Building LangGraph](https://blog.langchain.com/building-langgraph/)
- LangChain: [Runtime behind production deep agents](https://www.langchain.com/blog/runtime-behind-production-deep-agents)

### 论文和技术报告

- [AI Harness Engineering: A Runtime Substrate for Foundation-Model Software Agents](https://arxiv.org/abs/2605.13357)
- [Harness-Bench: Measuring Harness Effects across Models in Realistic Agent Workflows](https://arxiv.org/abs/2605.27922)
- [Agentic Harness Engineering: Observability-Driven Automatic Evolution of Coding-Agent Harnesses](https://arxiv.org/abs/2604.25850)
- [Meta-Harness: End-to-End Optimization of Model Harnesses](https://arxiv.org/abs/2603.28052)
- [Terminal-Bench: Benchmarking Agents on Hard, Realistic Tasks in Command Line Interfaces](https://arxiv.org/abs/2601.11868)
- [SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering](https://arxiv.org/abs/2405.15793)
- [OpenHands: An Open Platform for AI Software Developers as Generalist Agents](https://arxiv.org/abs/2407.16741)
- [Magentic-One: A Generalist Multi-Agent System for Solving Complex Tasks](https://arxiv.org/abs/2411.04468)
- [Magentic-UI: Towards Human-in-the-loop Agentic Systems](https://arxiv.org/abs/2507.22358)
- [Agent Lightning: Train ANY AI Agents with Reinforcement Learning](https://arxiv.org/abs/2508.03680)
- [Agentic Program Repair from Test Failures at Scale](https://arxiv.org/abs/2507.18755)
- [Building AI Coding Agents for the Terminal](https://arxiv.org/abs/2603.05344)
- [Agentic Workflow Injection](https://arxiv.org/abs/2605.07135)
- [Comment and Control: Hijacking Agentic Workflows via Context-Grounded Evolution](https://arxiv.org/abs/2605.11229)
- [Overeager Coding Agents](https://arxiv.org/abs/2605.18583)

## 最短总结

最新 harness engineering 的实践经验可以压缩成一句话：

> 把模型放进一个可复制、可观测、可验证、可权限控制、可持续改进的工程系统里；能确定的步骤用代码做，不确定的步骤给 agent 做；每次失败都回流成文档、工具、测试、权限或 eval 的改进。
