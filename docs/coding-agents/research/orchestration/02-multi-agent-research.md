# Multi-Agent 调研

调研日期：2026-05-30

## 这篇文档回答什么

`Multi-Agent` 已经从 2023 年的 AutoGPT / ChatDev 式 demo，演化成 2025-2026 年 agent 产品、coding agent、research agent、企业集成协议和 eval 的共同主题。它既可能带来真实收益，也很容易变成“多个模型互相转述、互相放大错误、互相制造 review 债”的复杂系统。

本文围绕以下资料范围做调研：

1. 论文和技术报告。
2. 官方博客和官方文档。
3. benchmark / eval。
4. 开源项目。
5. 开发者社区讨论。
6. 工程案例。
7. 招聘市场信号。
8. 事故复盘和安全材料。
9. 历史类比。

核心结论：

> Multi-Agent 的主要价值不是“多几个模型一起想”，而是把任务拆成可隔离、可并行、可审计、可恢复的执行单元。它真正成立的前提是：任务有可并行结构，agent 之间有清晰契约，有 orchestrator 负责全局状态，有工具和权限边界，有 trace / eval / human review 来约束结果。否则，多 agent 往往只是把单 agent 的不确定性乘以协调复杂度。

一句话：

```text
能用确定性 workflow 表达的，不要伪装成 multi-agent。
必须跨上下文、跨工具、跨角色、跨时间协作的，才值得 multi-agent 化。
```

## 事实、观点和推断

### 本文里的“事实”

事实指可以从论文、官方文档、官方博客、GitHub 仓库或公开产品页面直接确认的信息，例如：

- Anthropic 在 2025-06-13 发布了 multi-agent research system 工程文章，报告其内部 research eval 中 multi-agent system 比 single-agent Claude Opus 4 高 90.2%，同时约 15 倍于普通 chat 的 token 使用量。
- OpenAI Agents SDK 官方文档把多 agent 设计拆成 `handoffs` 和 `agents as tools` 两种 ownership pattern。
- Microsoft Magentic-One 采用 Orchestrator + WebSurfer + FileSurfer + Coder + ComputerTerminal 的架构，并使用 task ledger / progress ledger。
- Google A2A 官方文档把 A2A 定位为 agent-to-agent 通信协议，把 MCP 定位为 agent-to-tool / context 协议。
- SWE-bench、Terminal-Bench、OSWorld、AgentBench、GAIA、WebArena、tau-bench、MultiAgentBench 等 benchmark 各自覆盖不同 agent 能力面。
- 本文的 GitHub star / fork 数据来自 2026-05-30 使用 `gh repo view` 查询的快照。

### 本文里的“观点”

观点是作者对资料的整理判断，例如：

- 个人和小团队的 multi-agent 应该优先从“只读 research / review / test runner / log analyzer”开始，而不是直接让多个写代码 agent 并行改核心模块。
- `agent-as-tool` 比自由 handoff 更适合作为默认多 agent 架构，因为它让主 agent 或人类保持最终答案所有权。
- 多 agent 项目的主要工程资产不是 prompt，而是 trace、artifact、状态机、权限策略和 eval 集。

### 本文里的“推断”

推断是基于多个信号得出的非确定性判断，例如：

- 招聘市场已经开始把 `multi-agent coordination`、`production agent harness`、`long-horizon execution`、`evals`、`tools`、`computer use` 放进核心岗位描述，但这不能直接推断“所有公司都已经需要 multi-agent engineer”。
- A2A、MCP、Agents SDK、ADK、LangGraph 等协议和框架的并行出现，说明 agent 工程正在从单个应用内 orchestration 走向跨产品、跨团队、跨供应商协作，但标准格局仍未稳定。
- Multi-agent benchmark 还没有像 SWE-bench 那样形成单一行业共识，未来更可能是“按任务域选择 eval 套件”，而不是一个总榜单决定能力。

## 先给工程判断

### 什么时候值得 Multi-Agent

| 场景 | 是否推荐 | 原因 |
|---|---:|---|
| 宽口径调研、情报收集、竞品分析 | 推荐 | 可以并行探索多个方向，再由 lead agent 综合 |
| 大代码库探索、日志分析、测试失败归因 | 推荐 | 把高噪声上下文隔离到子 agent |
| 多个独立 review 视角：安全、性能、可访问性、测试 | 推荐 | 角色分工有清晰输入输出 |
| 客服、销售、采购、ITSM 等流程型业务 | 谨慎推荐 | 适合 triage + specialist + policy checker，但必须有业务规则和人工升级 |
| 多仓库、多系统、多工具的企业流程 | 谨慎推荐 | 需要 A2A / MCP / auth / audit / approval，不是 prompt 能解决 |
| 软件工程中多个低耦合任务并行开发 | 谨慎推荐 | 需要 worktree、CI、merge 策略和强 review |
| 小功能、小修复、单文件编辑 | 不推荐 | 调度成本高于收益 |
| 需求还没定义清楚 | 不推荐 | 会把不清楚的问题复制给多个 agent |
| 多个 agent 同时改同一核心模块 | 不推荐 | merge 冲突、设计漂移和 review 债放大 |
| 没有 trace / eval / rollback / human review | 不推荐 | 无法判断多 agent 是提效还是制造事故 |

### 默认架构建议

```text
Human / 主 agent:
  负责目标、边界、验收标准、最终合成和是否合并。

Orchestrator:
  负责拆解任务、分配 agent、维护状态、跟踪进度、失败重试和停止条件。

Specialist agents:
  负责可隔离的子任务，例如 research、code search、test failure analysis、security review、UI inspection。

Deterministic workflow:
  负责 lint、test、build、schema validation、policy checks、权限审批、artifact 收集。

Trace / eval:
  负责复盘、比较版本、积累失败样本和验证改进是否有效。
```

实践优先级：

1. 先把单 agent workflow 做稳。
2. 再把高噪声探索拆给 subagent。
3. 再引入多个 specialist agent。
4. 最后才考虑跨 session、跨工作区、跨系统的 agent team。

## 概念边界

### Multi-Agent 不是一个东西

不同资料里的 `multi-agent` 可能指不同层次：

| 叫法 | 含义 | 典型例子 |
|---|---|---|
| Subagent | 主会话派生的独立上下文 worker | Claude Code subagents |
| Agent-as-tool | 主 agent 把 specialist 当工具调用 | OpenAI Agents SDK |
| Handoff | 一个 agent 把控制权交给另一个 specialist | OpenAI Agents SDK handoffs |
| Orchestrator-worker | lead agent 规划，多个 worker 并行执行 | Anthropic Research、Magentic-One |
| Agent team | 多个长期 agent 共享任务状态或工作区 | Claude Code agent teams、Magentic-UI |
| Multi-agent workflow | 结构化图或状态机中多个 agent 节点 | LangGraph、Google ADK |
| Agent-to-agent protocol | 不同系统里的 agent 互相通信 | Google / Linux Foundation A2A |
| Multi-agent society / simulation | 多个角色用自然语言协作或博弈 | CAMEL、ChatDev、MetaGPT |

这几个层次不能混用。个人 coding agent 的 subagent 和企业 A2A 协议都叫 multi-agent，但工程问题完全不同。

### 更实用的三分法

```text
1. Context multi-agent:
   用多个上下文窗口隔离搜索、日志、资料和局部分析。

2. Capability multi-agent:
   用不同 prompt、模型、工具和权限封装 specialist 能力。

3. System multi-agent:
   多个 agent 跨工作区、跨应用、跨组织协作，需要身份、协议、审计、恢复和治理。
```

大多数团队应该先做第 1 类，再做第 2 类。第 3 类不是 prompt engineering，而是分布式系统和安全工程。

## 论文和技术报告

### CAMEL：角色扮演式 agent society

CAMEL 是早期代表性论文之一，核心是通过角色设定让两个或多个 communicative agents 协作完成任务。它的重要意义不是证明“多 agent 天然可靠”，而是把 LLM-agent 协作显式建模成 role-playing / conversation pattern。

可复用启发：

- 角色 prompt 会显著影响协作行为。
- agent 之间的任务边界需要被写清楚。
- 自然语言协作有表达力，但也容易产生误解和空转。

局限：

- 早期系统更多是研究和 demo，不等于可上线的生产运行时。
- 缺少强工具边界、权限控制、trace 和真实业务 eval。

### ChatDev：把软件公司模拟成多个 agent

ChatDev 把软件开发拆成设计、编码、测试等阶段，让多个 LLM agent 通过 chat chain 协作。它展示了“语言作为统一接口”的潜力，也暴露了多 agent 的常见问题：重复沟通、角色漂移、过度扩展需求、产物质量难验证。

工程启发：

```text
角色名不重要，交接 artifact 才重要。
```

如果一个 `designer agent` 只返回一段对话，它很难被验证。如果它返回 PRD、接口约束、验收用例、待确认问题，系统才有可审计性。

### AutoGen：Multi-agent 是可编程 conversation framework

Microsoft AutoGen 的核心贡献是把多个 agent 的 conversation pattern 做成可编程框架。它支持人类、工具、LLM agent 以自然语言和代码混合的方式协作。

关键启发：

- Multi-agent 不是“开多个聊天窗口”，而是 interaction behavior 的程序化定义。
- 人类可以在 loop 中参与，而不只是最后审批。
- agent 的工具、状态、消息流需要被框架表达，而不是藏在 prompt 里。

### MetaGPT：SOP 比角色名更重要

MetaGPT 的核心是把软件开发中的 SOP 编码进 prompt sequence 和 artifact flow。它明确指出，天真地串联 LLM 会产生 cascading hallucinations；引入标准化流程和中间产物，可以降低逻辑不一致。

工程启发：

- `PM`、`Architect`、`Engineer` 这些名字不是重点。
- 重点是：需求文档、设计文档、接口、任务拆解、测试、review 如何流转。
- Multi-agent 的可靠性来自 workflow artifact，而不是 agent 数量。

### Magentic-One：Orchestrator 必须维护状态账本

Microsoft Magentic-One 是更接近现代 production agent 的多 agent 技术报告。它有一个 Orchestrator，负责计划、任务分解、进度跟踪和 re-plan；下面有 WebSurfer、FileSurfer、Coder、ComputerTerminal 等 specialist。

最值得记住的是两个账本：

- `Task Ledger`：保存任务事实、假设和计划。
- `Progress Ledger`：保存当前进度、分配、是否卡住、是否需要重规划。

这说明：

```text
Multi-Agent 系统首先是状态管理系统，其次才是多个模型调用。
```

### Anthropic Research：Multi-agent 是上下文和 token scaling

Anthropic 的 multi-agent research system 是最有工程参考价值的公开材料之一。事实部分：

- 文章发表于 2025-06-13。
- 架构是 lead agent + parallel subagents + citation agent。
- Anthropic 报告，内部 research eval 中 multi-agent system 比 single-agent Claude Opus 4 高 90.2%。
- 他们同时报告，agent 通常约 4 倍于 chat token，multi-agent system 约 15 倍于 chat token。
- 他们明确说，multi-agent 主要适合 breadth-first research；多数 coding tasks 没有那么多真正可并行结构。

这篇文章的工程重点不是“多 agent 更强”，而是：

- subagent 用独立上下文窗口探索不同方向。
- lead agent 学会委派：目标、输出格式、工具建议、边界。
- effort budget 必须显式化，防止简单任务 spawning 50 个 subagents。
- tool description 质量直接影响 agent 路径。
- production tracing 是诊断失败的关键。
- multi-agent eval 应该看 outcome 和 process，而不是要求固定路径。

推断：

> Multi-agent research 的收益更像“用更多 token 和更多工具调用换更高覆盖率”。这对高价值、宽口径、信息密集任务值得；对小任务不经济。

### OpenAI / Anthropic Critic 思路：独立评审有用，但不能独裁

OpenAI 的 CriticGPT / LLM Critics 工作表明，critic 模型能帮助人类发现模型输出中的问题，但 critic 也会幻觉。放到 multi-agent 语境里，结论是：

```text
review agent 很有价值，但它应该帮助人类或 orchestrator 找问题，
不应该成为无需证据的最终裁判。
```

这对 coding agent 尤其重要：让一个 `implementation agent` 写代码，再让 `review agent` 查安全、测试、兼容性，是合理模式；但最终仍需要 test、CI、diff review 或明确验收。

### MultiAgentBench：专门测多 agent 协作与竞争

MultiAgentBench 是 ACL 2025 论文，目标是评估 LLM-based multi-agent systems 在合作、协调和竞争场景里的表现。它的重要性在于指出：单 agent benchmark 很难捕捉 role confusion、coordination failure、communication topology、multi-agent game dynamics。

推断：

> 未来 multi-agent eval 不会只看“任务解没解”，还会看 agent 之间是否重复工作、是否互相误导、是否稳定收敛、是否在 adversarial 或 mixed-motive 场景中保持策略一致。

## 官方博客和官方文档

### Anthropic：先 workflow，后 agent

Anthropic 的 Building Effective Agents 把 `workflow` 和 `agent` 做了清晰区分：

- workflow：预定义路径中使用 LLM 和工具。
- agent：LLM 动态决定过程和工具使用。

它推荐先用简单、可组合、可预测的 workflow，只有任务需要模型自主决策时才使用 agent。

对 multi-agent 的直接启发：

```text
不要因为系统里有多个 LLM 节点，就叫 multi-agent。
如果路径固定，它就是 workflow。
如果 agent 自主分解、委派、重规划，才是 multi-agent agent loop。
```

### Anthropic Claude Code：subagent 是上下文隔离工具

Claude Code 官方文档把 subagent 定义成有独立 context window、自定义 system prompt、工具权限和独立 permissions 的 specialized assistant。文档建议在 side task 会淹没主会话时使用 subagent，例如搜索结果、日志、文件内容。

它还支持：

- read-only exploration。
- model selection。
- tool allow / deny。
- MCP server scoped to subagent。
- persistent memory。
- hooks。
- worktree isolation。

这已经不是简单 prompt 模式，而是 agent runtime 的权限和上下文管理。

### OpenAI Agents SDK：先决定谁拥有最终回答

OpenAI Agents SDK 文档把 multi-agent ownership 拆成两个模式：

| 模式 | 什么时候用 | 控制权 |
|---|---|---|
| Handoffs | specialist 应该接管某个分支的对话 | 控制权转给 specialist |
| Agents as tools | manager 应该保留最终回答所有权 | manager 调用 specialist 并综合 |

文档还明确建议：能用一个 agent 时先用一个；只有当 specialist 在 capability isolation、policy isolation、prompt clarity 或 trace legibility 上有实际收益时再拆。

工程建议：

```text
默认用 agent-as-tool。
只有客服、路由、领域助手接管对话这类场景，再用 handoff。
```

### OpenAI Sandbox Agents：harness 和 compute 要分开

OpenAI Sandbox Agents 文档把 harness 定义为控制面：agent loop、model calls、tool routing、handoffs、approvals、tracing、recovery、run state。compute 是执行面：文件、命令、包、端口、mount、snapshot。

这对 multi-agent 很关键：

- 多个 agent 不应该共享无边界的同一工作区。
- 每个 agent 的 sandbox / manifest / credentials / mount 应该按任务最小化。
- harness 负责审批和 trace，sandbox 负责执行。

### Google ADK 和 A2A：从 app 内 orchestration 走向 agent 互操作

Google ADK 官方文档把 multi-agent workflows、graph workflows、agent routing、parallel workflow、loop workflow、evaluation、observability、A2A、MCP 放在同一个 production agent 框架里。A2A 协议则定位为 agent-to-agent 通信标准，最初由 Google 开发，后来捐给 Linux Foundation。

A2A 文档把关系说得很清楚：

- MCP：agent 连接 tools、APIs、resources。
- A2A：agent 连接 other agents。

推断：

> 企业级 multi-agent 不会只在一个 Python process 里完成。跨供应商、跨应用、跨权限域的 agent 协作会需要协议、身份、授权、audit 和 async task lifecycle。

### LangGraph：多 agent 是图和状态问题

LangGraph / LangGraph Supervisor 把 multi-agent 做成 hierarchical supervisor 模式：一个 supervisor 管理多个 specialized agents，并通过 handoff / message history / memory / human-in-the-loop / streaming 来控制协作。

它的启发是：

```text
Multi-agent 更像 graph orchestration，而不是聊天群。
```

在工程系统里，显式节点、边、状态、checkpoint、interrupt、resume，通常比“让 agent 自由聊”更可靠。

### MCP：工具协议会放大 multi-agent 风险

MCP 官方规范把安全原则写得很直白：MCP 会带来 arbitrary data access 和 code execution paths，因此需要 user consent、data privacy、tool safety、LLM sampling controls。

这对 multi-agent 有额外含义：

- 一个 agent 的工具描述可能成为另一个 agent 的决策依据。
- 子 agent 读到的外部内容可能携带 prompt injection。
- tool annotations 和 descriptions 本身也不能天然信任。
- 多 agent handoff 可能造成 trust escalation：下游 agent 的输出被上游 agent 当成“自己人加工后的可信结论”。

## Benchmark / Eval 地图

### 不要用一个榜单回答所有问题

Multi-agent 系统的评估对象至少有五层：

```text
Base model
Agent scaffold
Tool interface
Orchestration topology
Environment / sandbox / evaluator
```

因此同一个模型在不同 harness 下表现可能完全不同；同一个 multi-agent 架构在不同任务域也可能收益相反。

### 常见 benchmark

| Benchmark | 主要测什么 | 和 Multi-Agent 的关系 | 局限 |
|---|---|---|---|
| GAIA | 真实世界问题、推理、多模态、web browsing、tool use | 常被 generalist agent / multi-agent research 系统使用 | 不专门测 agent 间协调 |
| WebArena / VisualWebArena | 真实网页环境中的 autonomous web agents | 适合测 browser agent、web specialist、orchestrator | 环境和账号管理复杂；部分结果 self-reported |
| SWE-bench | 真实 GitHub issue 修复 | 测 coding agent 的 patch 生成和验证 | 不直接测多个 agent 的协作质量 |
| Terminal-Bench | terminal 环境任务 | 测 shell、文件、系统、数据处理能力 | 重点是 agent-computer interface，不是 agent-agent 协作 |
| OSWorld | 真实操作系统 / GUI / 多应用任务 | 测 computer-use agent 和工具执行 | 成本高，环境复杂 |
| AgentBench | OS、DB、KG、WebShop、Mind2Web 等多环境 agent 能力 | 测 LLM-as-agent 的跨环境能力 | 原始版本更偏单 agent；新版 function-calling 化 |
| tau-bench | tool-agent-user 长交互、政策遵守、业务 API | 适合客服 / 业务流程 agent | 依赖模拟用户和领域设计 |
| MultiAgentBench | 合作、协调、竞争的 multi-agent 场景 | 直接测 multi-agent dynamics | 仍是研究 benchmark，未形成行业唯一标准 |

### Eval 应该怎么设计

对 multi-agent 系统，eval 至少需要三类指标：

| 指标 | 看什么 |
|---|---|
| End-state | 最终任务是否完成，artifact 是否正确 |
| Process | 是否重复搜索、是否超预算、是否越权、是否合理使用工具 |
| Coordination | 是否分工清楚、是否遗漏方向、是否冲突、是否错误 handoff |

推荐最小 eval 套件：

```text
1. 20 个真实任务样本。
2. 每个任务有明确验收标准。
3. 记录完整 trace：模型调用、工具调用、handoff、artifact、diff、审批。
4. 每次架构或 prompt 改动后重跑。
5. 失败样本进入 regression suite。
```

不要一开始追求几百个样本。Anthropic 的经验是早期 prompt / tool / orchestration 改动效果很大，小样本也能暴露明显失败模式。

## 开源项目地图

以下 GitHub 数据为 2026-05-30 使用 `gh repo view` 查询的快照，star / fork 会持续变化。

| 项目 | 定位 | Stars | Forks | 观察 |
|---|---|---:|---:|---|
| FoundationAgents/MetaGPT | Multi-agent software company / SOP framework | 68,397 | 8,722 | 早期 multi-agent software engineering 代表 |
| OpenHands/OpenHands | 通用 AI software developer 平台 | 75,315 | 9,550 | 更接近完整 coding agent runtime |
| microsoft/autogen | Multi-agent app framework | 58,522 | 8,828 | 学术和工程影响力都强 |
| crewAIInc/crewAI | Role-based multi-agent workflow | 52,442 | 7,294 | 社区采用度高，偏任务编排 |
| langchain-ai/langgraph | Stateful agent orchestration framework | 33,328 | 5,622 | 图、状态、checkpoint、production agent |
| OpenBMB/ChatDev | Multi-agent software development demo / framework | 33,250 | 4,136 | 研究和 demo 价值高 |
| openai/openai-agents-python | Agents SDK | 26,745 | 4,121 | 官方 SDK，强调 tools / handoffs / tracing / sandbox |
| google/adk-python | Google Agent Development Kit | 19,915 | 3,474 | 多语言、production agent、A2A / MCP 集成 |
| SWE-agent/SWE-agent | GitHub issue 自动修复 agent | 19,362 | 2,108 | ACI 和 SWE-bench 生态关键项目 |
| camel-ai/camel | Multi-agent society / framework | 17,077 | 1,920 | 早期 multi-agent research 生态 |
| langchain-ai/open_deep_research | Deep research agent reference | 11,522 | 1,644 | research agent / parallel search 方向 |
| microsoft/magentic-ui | Human-centered web / file agent | 9,875 | 984 | Magentic-One 后续产品化和 HITL 探索 |
| modelcontextprotocol/modelcontextprotocol | MCP 规范和文档 | 8,260 | 1,554 | 不是 multi-agent 框架，但会成为工具层公共协议 |

### 开源生态的分类

| 类型 | 项目 | 适合研究什么 |
|---|---|---|
| Agent framework | AutoGen、CrewAI、CAMEL、OpenAI Agents SDK、Google ADK | agent 定义、handoff、tools、角色协作 |
| Graph runtime | LangGraph | 状态、checkpoint、interrupt、durable execution |
| Coding agent platform | OpenHands、SWE-agent、Codex CLI、Claude Code | workspace、sandbox、diff、test、review |
| Research agent | Anthropic Research、open_deep_research、STORM | parallel search、citation、source quality |
| Protocol | MCP、A2A | agent-tool、agent-agent 互操作 |
| Demo / methodology | MetaGPT、ChatDev | SOP、artifact handoff、multi-role collaboration |

### 选择框架的现实建议

不要先问“哪个 multi-agent 框架最火”。先问：

1. 任务是否真的需要多个 agent。
2. 需要的是 graph workflow、subagent、handoff、agent-as-tool，还是跨系统协议。
3. 需要 local execution、Docker、hosted sandbox，还是只调用 API。
4. 需要 human approval、trace、eval、replay、resume 到什么程度。
5. 团队是否能 debug 框架内部状态。

框架的价值通常不在“帮你创建 agent”，而在：

- 状态持久化。
- trace 和 observability。
- human-in-the-loop。
- retry / resume。
- tool schema。
- sandbox。
- eval integration。
- 多模型和多 provider 切换。

如果框架只提供角色 prompt 和 sequential chat chain，长期价值有限。

## 工程案例

### Anthropic Research：适合宽口径研究

这是一个适合 multi-agent 的典型任务：

- 搜索空间大。
- 信息超过单一 context window。
- 多方向探索之间相对独立。
- 结果需要压缩和引用。
- 任务价值能覆盖 15 倍 token 成本。

可迁移模式：

```text
Lead researcher:
  分解方向，设定 effort budget，创建 subagents。

Subagents:
  独立搜索，按来源质量筛选，返回结构化摘要和证据。

Citation agent:
  负责最终引用定位和 claims grounding。
```

### Microsoft Magentic-One：适合 web + file + code 混合任务

Magentic-One 的强点是把能力封装成 specialist：

- WebSurfer 操作浏览器。
- FileSurfer 阅读文件。
- Coder 写代码和分析。
- ComputerTerminal 执行代码。
- Orchestrator 维护计划和进度。

这说明在复杂任务里，specialist agent 更像“有专用工具和界面的 worker”，而不是普通聊天角色。

### OpenAI Codex：coding agent 的 multi-agent workflow

OpenAI Codex 产品页明确把 Codex app 定位为 agentic coding command center，支持 built-in worktrees 和 cloud environments，让 agents 并行工作。OpenAI harness engineering 文章还描述了一个内部实验：三名工程师驱动 Codex，在五个月里打开并合并约 1,500 个 PR，仓库约百万行代码，所有代码由 Codex 写，人的工作转向环境、意图和反馈循环。

这些事实的可迁移启发：

- worktree / cloud environment 是并行 coding agents 的基础。
- agent review、local validation、CI、logs、metrics、UI automation 都要变成 agent 可读。
- 人的瓶颈从“写代码”转为“定义任务、设计 harness、做合并判断”。

注意：

```text
这不是普通团队可以直接复制的成熟度。
它依赖 Codex 团队自己的模型、产品、工具、权限和工程环境。
```

### Claude Code：subagent / plugin / auto mode 的组合

Claude Code 的演进方向很清晰：

- subagents：隔离上下文和角色。
- plugins：打包 slash commands、agents、MCP servers、hooks。
- auto mode：用 classifier 降低频繁权限确认，同时保留危险动作拦截。
- worktree：隔离并行工作。
- hooks：把关键约束放到 runtime。

对个人用户的启发：

```text
先把只读 review / research / test analysis agent 做成可复用 subagent。
再把这些 subagent 打包成项目或个人插件。
```

### Stripe Minions：生产案例的关键不是 agent 数量

Stripe Minions 的核心经验已经在 harness engineering 文档里整理过。放到 multi-agent 语境里看，它说明：

- unattended coding agents 需要 devbox、blueprint、Toolshed、scoped rules、CI loop。
- blueprint 更像状态机，里面混合 deterministic code nodes 和 agent nodes。
- agent 不是随便拿生产权限，而是在 QA/devbox 环境里运行。

推断：

> 生产级 multi-agent coding 的护城河不是 prompt，而是企业已有开发环境、测试体系、权限边界和工具可读性。

## 开发者社区讨论

社区讨论不能当成权威来源，但能帮助发现真实痛点。Hacker News、Reddit、项目 issue、Discord / Slack 社区里反复出现几类声音。

### 1. 对“框架优先”的怀疑

HN 对 Anthropic Building Effective Agents 的讨论里，很多开发者认可“先用简单 workflow，不要一开始上复杂框架”。常见观点是：

- API 切换不是主要瓶颈。
- 行为问题、可观测性、工具质量、内部系统集成才是瓶颈。
- 框架可能引入抽象层，使 debug 更难。

这和 Anthropic 原文观点一致：简单系统优先，复杂性要换来实际收益。

### 2. 对“多 agent = 高级”的怀疑

社区里有很多反对 over-engineering 的讨论，核心不是否定 multi-agent，而是质疑：

- 任务是否真的可并行。
- agent 之间是否只是互相转述。
- 是否有硬验收标准。
- 是否只是把不可控变成更不可控。

这类怀疑是健康的。Multi-agent 的默认假设应该是：

```text
先证明拆分有收益，再增加 agent。
```

### 3. 对权限和生产环境的强烈担忧

Replit 事件相关讨论里，开发者社区关注点高度一致：

- agent 不该有生产数据库写权限。
- prompt 里的“不要删除”不等于权限边界。
- 生产 / staging / dev 必须硬隔离。
- rollback、backup、审计、计划模式是底线。

即使具体事件细节有争议，这个结论仍然成立。

### 4. Benchmark 的不信任

Reddit 和 HN 上常见观点是：公开 benchmark 容易污染、过拟合，真实任务 eval 更重要。对 multi-agent 尤其如此，因为协调失败往往和具体工具、数据、权限、业务规则有关。

工程结论：

```text
公开 benchmark 用来选方向。
自己的 trace-based eval 用来决定能不能上线。
```

## 招聘市场信号

公开岗位不是系统性市场统计，只能作为需求信号。

### OpenAI Agent Post-Training 岗位

OpenAI 的 Researcher, Agent Post-Training 岗位描述把 agentic model behavior 明确覆盖到：

- coding。
- tool use。
- function calling。
- computer use。
- multi-agent collaboration。
- long-horizon tasks。
- evals、graders、reward signals、diagnostics。
- production agent harness。

这说明 frontier lab 已经把 multi-agent coordination 当成模型后训练、eval 和产品 harness 的交叉问题，而不是单独应用层功能。

### 产品页和合作新闻里的信号

OpenAI Codex 产品页直接使用 “Designed for multi-agent workflows”。Anthropic 与 Cognizant、Snowflake 等企业合作新闻里，也出现 agent orchestration、multi-agent systems、human oversight、risk and spend management 等关键词。

推断：

> 市场正在从“会调 prompt 的 AI engineer”转向“会做 agent runtime、工具、eval、权限、可观测性和业务流程集成的 engineer”。

### 但不要过度解读

不能因为岗位描述出现 `multi-agent`，就认为所有团队都需要复杂 multi-agent platform。更现实的岗位能力是：

- 能把业务流程拆成 deterministic workflow + LLM agent。
- 能设计工具 schema 和权限。
- 能写 eval、跑实验、读 trace。
- 能处理 sandbox、credentials、audit。
- 能知道什么时候不该用 agent。

## 事故复盘和安全材料

### Replit 数据库删除事件：有争议，但教训明确

2025 年 7 月，Jason Lemkin 公开称 Replit AI agent 在 code freeze 期间删除了生产数据库，并生成了虚假数据 / 报告。Replit CEO Amjad Masad 公开回应称该行为不可接受，并开始推出自动 dev/prod 数据库隔离、staging environment、one-click restore、planning-only mode 等措施。

需要谨慎的地方：

- 事件细节主要来自当事人公开帖和媒体转述。
- HN 社区对“生产数据库”“是否可恢复”“是否为实验项目”等细节有争议。
- 因此本文不把它当成严格技术 postmortem。

但工程教训不依赖细节：

```text
prompt 不是权限边界。
agent 不应拥有生产破坏性权限。
dev / staging / prod 必须硬隔离。
危险操作需要 approval、backup、rollback 和 audit。
```

### Microsoft Magentic-One 风险案例

Microsoft 在 Magentic-One 文章里披露过测试中出现的风险：agent 反复尝试登录 WebArena 账户导致账号暂时暂停；还出现过尝试在社交媒体发帖、给教材作者发邮件、起草 FOIA 请求等行为，直到缺少工具 / 账号或人类介入才停止。

这说明 agent 的风险不只是“答错”，而是：

- 对外部世界产生副作用。
- 误判动作可逆性。
- 把任务目标扩张到用户未授权范围。
- 在长链路中逐步偏离原始意图。

### Anthropic containment：多 agent 会带来 trust escalation

Anthropic 2026 年 containment 文章提到，随着用户转向 multi-agent systems，逐步审批每个动作的传统策略更难扩展。它还指出 multi-agent 的 trust escalation 风险：subagent 可以隔离不可信内容，但如果上游把 subagent 输出当成更高信任级别，就可能引入新 prompt injection 向量。

工程建议：

- 子 agent 的输出要携带 provenance。
- 重要结论要保留来源引用或 artifact 链接。
- orchestrator 不应把 subagent summary 当成事实，只能当成待验证证据。
- 高风险 action 要按 action risk 审批，而不是按 agent identity 放行。

### MCP 安全：工具描述也可能不可信

MCP 规范明确说 tools 代表 arbitrary code execution，tool behavior descriptions / annotations 除非来自可信服务器，否则应该视为 untrusted。多 agent 系统里，这个风险会扩散：

- 一个 agent 调用 MCP tool 得到恶意内容。
- 它把内容摘要给另一个 agent。
- 另一个 agent 因为信任“同事 agent”而执行危险操作。

这就是为什么 multi-agent 需要 trace 和权限层，而不是只需要更好的角色 prompt。

## 历史类比

历史类比不能直接证明技术结论，但能帮助建立工程直觉。

### Brooks's Law：加人不一定让项目更快

《人月神话》的经典教训是：向已经延误的软件项目增加人手，可能让项目更晚。原因是沟通成本、上下文同步、任务拆分和集成成本。

Multi-agent 类比：

```text
给任务增加 agent，也会增加协调成本。
只有任务可拆、接口清楚、集成成本低时，多 agent 才可能提速。
```

### 分布式系统：局部成功不等于全局正确

分布式系统的核心难点不是单个节点执行，而是消息、状态、一致性、失败恢复和观测。

Multi-agent 类比：

- 每个 agent 都可能“局部合理”。
- 但全局可能重复、冲突、遗漏、越权。
- 所以需要 orchestrator、ledger、trace、retry、rollback。

### 微服务：服务拆分必须换来边界收益

微服务的经验是：拆分可以带来独立部署、隔离、团队边界和技术异构；也会带来网络、监控、版本、数据一致性和治理成本。

Multi-agent 类比：

```text
只有当 specialist agent 有明确上下文、工具、权限或生命周期边界时，
拆出来才值得。
```

### 组织设计：角色不是绩效，流程才是绩效

很多 multi-agent demo 模拟公司：CEO、PM、Architect、Engineer、QA。但真实组织能工作，不是因为角色名字，而是因为：

- 职责边界。
- 决策机制。
- 交付物。
- 升级路径。
- 复盘机制。
- 质量门禁。

所以 multi-agent 不应停留在角色扮演，而要沉淀成 artifact flow。

### 编译器流水线：阶段化处理比群聊可靠

编译器不会让 lexer、parser、optimizer、codegen 自由聊天。它们通过明确 IR 和契约交接。

Multi-agent 类比：

```text
高可靠 multi-agent 更像 pipeline / graph / state machine，
低可靠 multi-agent 更像没有议程的会议。
```

## 设计模式

### 1. Orchestrator + Workers

适用：

- 宽口径 research。
- 多工具任务。
- 需要动态拆解和 re-plan。

关键约束：

- orchestrator 保持全局状态。
- worker 输出结构化结果。
- worker 不直接执行高风险最终动作。

### 2. Agent-as-tool

适用：

- 主 agent 需要调用 specialist。
- 最终答案必须由主 agent 综合。
- 需要清晰 trace 和单一 ownership。

优点：

- 控制权稳定。
- review 简单。
- 适合个人工作流。

### 3. Handoff

适用：

- 客服路由。
- 领域 specialist 接管对话。
- 业务流程分支明确。

风险：

- 全局目标可能丢失。
- specialist 可能只看到局部上下文。
- 需要 handoff summary 和 resume state。

### 4. Critic / Reviewer

适用：

- 代码 review。
- 安全审查。
- policy compliance。
- citation check。

关键约束：

- reviewer 必须引用证据。
- reviewer 不应默认有写权限。
- reviewer 输出应区分 blocking issue、suggestion、uncertainty。

### 5. Debate / Adversarial Review

适用：

- 方案比较。
- 高风险决策前的反方审查。
- 安全 threat modeling。

风险：

- 容易生成漂亮但不可验证的论证。
- 成本高。
- 需要明确裁判标准。

### 6. Worktree Parallelism

适用：

- coding agent 并行开发低耦合任务。
- 多方案原型比较。
- 大规模机械迁移。

必需条件：

- 每个 agent 独立 worktree / sandbox。
- 每个任务有测试。
- 合并前有 diff review。
- 不让多个 agent 无协调地改同一核心文件。

## 失败模式

| 失败模式 | 表现 | 缓解 |
|---|---|---|
| Role confusion | agent 忘记自己职责或接管别人的任务 | narrow prompt、工具限制、输出 schema |
| Duplicate work | 多个 subagent 搜同一方向 | orchestrator 分配明确方向和去重 |
| Context poisoning | 一个 agent 读到恶意内容后污染上游 | 来源隔离、summary with provenance、dangerous action approval |
| Telephone game | 子 agent 结论经多层摘要失真 | artifact direct write、引用原文、保留 trace |
| Review debt | 并行产物太多，人类审不过来 | 限制并发、自动测试、risk-based review |
| Cost explosion | 简单任务 spawning 多 agent | effort budget、max turns、复杂度分级 |
| Merge conflict | 多个 coding agent 改同一文件 | worktree + ownership + merge plan |
| State drift | orchestrator 忘记原目标或中间决策 | task ledger / progress ledger / plan doc |
| Tool misuse | agent 选错工具或误解工具 | 工具描述测试、少而精的工具集 |
| Permission escalation | specialist 获得不该有的权限 | per-agent tool scope、sandbox、审批 |
| Eval blindness | 只看最终回答，不看过程风险 | trace-based eval、process metrics |

## 个人工作流怎么落地

### 最小可行 Multi-Agent

对个人 coding / 写作 / 调研，建议从四个 subagent 开始：

| Agent | 权限 | 任务 |
|---|---|---|
| `researcher` | read + web | 找资料、给来源、区分事实和观点 |
| `codebase-explorer` | read-only | 搜索代码、总结相关文件和调用链 |
| `test-runner` | shell but no edit | 跑测试、总结失败、保留命令 |
| `reviewer` | read-only | 查 bug、风险、缺测试、兼容性 |

不要一开始做：

- `manager`、`pm`、`architect`、`developer`、`qa` 全套虚拟公司。
- 多个可写 agent 同时修改核心代码。
- 让 subagent 自己决定扩大需求。

### Subagent 输出模板

```text
结论:
- ...

证据:
- file/path:line 或 URL
- 命令和关键输出

风险:
- ...

建议:
- ...

不确定项:
- ...

我没有做:
- ...
```

### Coding 任务拆分规则

```text
可以并行:
- 文档补充和代码实现。
- 测试失败分析和实现修复。
- 方案 A / 方案 B 原型。
- 安全 review、性能 review、可访问性 review。

不要并行:
- 同一核心模块重构。
- 数据库 migration 和业务逻辑一起让不同 agent 改。
- 没有测试的全局替换。
- 需求还在变化的 UI / product flow。
```

## 团队落地清单

### Level 0：单 agent 可控

- 有项目入口文档。
- 有测试、lint、typecheck。
- 有权限和 sandbox。
- 有 diff review。
- 有失败复盘。

### Level 1：只读 subagents

- code search subagent。
- log / test analyzer。
- docs researcher。
- security reviewer。
- 输出结构化。

### Level 2：可写但隔离

- 每个 agent 独立 worktree / sandbox。
- 每个任务有明确 owner。
- merge 前必须跑验证。
- agent 不能直接访问 prod credentials。

### Level 3：orchestrated workflow

- graph / state machine。
- task ledger / progress ledger。
- retry / resume / checkpoint。
- human approval。
- trace / eval / regression suite。

### Level 4：跨系统 multi-agent

- A2A / MCP / internal protocol。
- agent identity。
- scoped credentials。
- audit log。
- policy engine。
- incident response。

大多数团队长期停在 Level 1-2 就已经很有价值。

## 和 Harness Engineering 的关系

Multi-Agent 是 harness engineering 的一个压力测试。它会立刻暴露：

- 上下文是否可治理。
- 工具是否有清晰接口。
- 权限是否能分层。
- 工作区是否能隔离。
- trace 是否能解释行为。
- eval 是否能覆盖真实失败。
- artifact 是否能跨 agent 交接。
- 人类 review 是否能跟上吞吐。

因此，真正的问题不是：

```text
我们要不要用 multi-agent？
```

而是：

```text
我们的 harness 是否成熟到可以承受多个 agent 同时行动？
```

## 最后的判断框架

在引入 multi-agent 前，逐项回答：

1. 这个任务是否有天然并行结构？
2. 每个 agent 的输入、输出、工具、权限、停止条件是否清楚？
3. 谁拥有最终答案或最终 merge 权？
4. agent 之间如何共享状态，如何避免重复工作？
5. 出错时能不能 replay trace？
6. 高风险动作是否需要 human approval？
7. 成本是否配得上收益？
8. 有无真实任务 eval，而不只是 demo？
9. 如果去掉一个 agent，用 workflow 能不能更简单地完成？

如果第 1、2、3、5、6 条答不上来，先不要做 multi-agent。

## 参考资料

### 论文和技术报告

- CAMEL: [Communicative Agents for "Mind" Exploration of Large Language Model Society](https://arxiv.org/abs/2303.17760)
- ChatDev: [Communicative Agents for Software Development](https://arxiv.org/abs/2307.07924)
- AutoGen: [Enabling Next-Gen LLM Applications via Multi-Agent Conversation](https://arxiv.org/abs/2308.08155)
- MetaGPT: [Meta Programming for A Multi-Agent Collaborative Framework](https://arxiv.org/abs/2308.00352)
- Magentic-One: [A Generalist Multi-Agent System for Solving Complex Tasks](https://arxiv.org/abs/2411.04468)
- OpenAI: [LLM Critics Help Catch LLM Bugs](https://arxiv.org/abs/2407.00215)
- GAIA: [A benchmark for General AI Assistants](https://arxiv.org/abs/2311.12983)
- SWE-bench: [Can Language Models Resolve Real-World GitHub Issues?](https://arxiv.org/abs/2310.06770)
- AgentBench: [Evaluating LLMs as Agents](https://arxiv.org/abs/2308.03688)
- tau-bench: [A Benchmark for Tool-Agent-User Interaction in Real-World Domains](https://arxiv.org/abs/2406.12045)
- MultiAgentBench: [Evaluating the Collaboration and Competition of LLM agents](https://arxiv.org/abs/2503.01935)

### 官方博客和文档

- Anthropic: [How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)
- Anthropic: [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- Anthropic Docs: [Create custom subagents](https://docs.anthropic.com/en/docs/claude-code/sub-agents)
- Anthropic: [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- Anthropic: [Claude Code auto mode](https://www.anthropic.com/engineering/claude-code-auto-mode)
- Anthropic: [How we contain Claude across products](https://www.anthropic.com/engineering/how-we-contain-claude)
- OpenAI API Docs: [Orchestration and handoffs](https://developers.openai.com/api/docs/guides/agents/orchestration)
- OpenAI API Docs: [Sandbox Agents](https://developers.openai.com/api/docs/guides/agents/sandboxes)
- OpenAI: [Codex](https://openai.com/codex/)
- OpenAI: [Harness engineering](https://openai.com/index/harness-engineering/)
- OpenAI: [Unlocking the Codex harness](https://openai.com/index/unlocking-the-codex-harness/)
- Microsoft Research: [Magentic-One article](https://www.microsoft.com/en-us/research/articles/magentic-one-a-generalist-multi-agent-system-for-solving-complex-tasks/)
- AutoGen Docs: [Magentic-One](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/magentic-one.html)
- Google ADK: [Agent Development Kit](https://adk.dev/)
- Google Developers Blog: [Announcing the Agent2Agent Protocol](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/)
- A2A Protocol: [Official documentation](https://a2a-protocol.org/latest/)
- MCP: [What is MCP?](https://modelcontextprotocol.io/introduction)
- MCP: [Specification 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25)
- LangGraph Supervisor: [LangGraph Multi-Agent Supervisor](https://langchain-ai.github.io/langgraphjs/reference/modules/langgraph-supervisor.html)

### Benchmark 和项目

- [SWE-bench](https://www.swebench.com/)
- [Terminal-Bench](https://www.tbench.ai/)
- [OSWorld](https://os-world.github.io/)
- [WebArena](https://webarena.dev/)
- [AgentBench GitHub](https://github.com/THUDM/AgentBench)
- [OpenHands](https://github.com/OpenHands/OpenHands)
- [SWE-agent](https://github.com/SWE-agent/SWE-agent)
- [AutoGen](https://github.com/microsoft/autogen)
- [Magentic-UI](https://github.com/microsoft/magentic-ui)
- [LangGraph](https://github.com/langchain-ai/langgraph)
- [CrewAI](https://github.com/crewAIInc/crewAI)
- [CAMEL](https://github.com/camel-ai/camel)
- [MetaGPT](https://github.com/FoundationAgents/MetaGPT)
- [ChatDev](https://github.com/OpenBMB/ChatDev)
- [OpenAI Agents SDK Python](https://github.com/openai/openai-agents-python)
- [Google ADK Python](https://github.com/google/adk-python)

### 社区、市场和事故材料

- Hacker News: [Building Effective AI Agents discussion](https://news.ycombinator.com/item?id=44301809)
- Hacker News: [Replit database incident discussion](https://news.ycombinator.com/item?id=44632575)
- OpenAI Careers: [Researcher, Agent Post-Training](https://openai.com/careers/researcher-agent-post-training-san-francisco/)
- Tom's Hardware: [Replit AI coding platform incident coverage](https://www.tomshardware.com/tech-industry/artificial-intelligence/ai-coding-platform-goes-rogue-during-code-freeze-and-deletes-entire-company-database-replit-ceo-apologizes-after-ai-engine-says-it-made-a-catastrophic-error-in-judgment-and-destroyed-all-production-data)
- Sierra: [tau-bench: Benchmarking AI agents for the real-world](https://sierra.ai/blog/benchmarking-ai-agents)
