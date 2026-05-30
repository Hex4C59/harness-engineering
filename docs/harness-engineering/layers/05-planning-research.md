# Agent Planning 调研

调研日期：2026-05-30

## 这篇文档回答什么

这篇文档围绕 AI agent 语境里的 **Planning** 做横向调研。这里的 Planning 不是单纯项目管理，也不只是 prompt 里让模型“先想一想”，而是 agent harness 中的一组运行时能力：

```text
Planning = 目标澄清 + 上下文获取 + 任务分解 + 工具选择 + 权限控制 + 执行顺序 + 进度跟踪 + 失败后重规划
```

资料范围包括论文、博客、技术报告、官方文档、benchmark / eval、开源项目、开发者社区讨论、工程案例、招聘市场、事故复盘和历史类比。涉及最新工具、岗位和产品能力的内容，以 2026-05-30 可查公开资料为准。

核心结论：

> Planning 正在从“模型内部推理技巧”变成 agent runtime 的控制面。可靠系统不会只要求模型生成一个自然语言计划，而是会把计划外化成可审查、可执行、可回放、可打断、可重规划的状态结构。

## 事实、观点和推断

本文刻意区分三类内容：

| 类型 | 本文含义 | 例子 |
|---|---|---|
| 事实 | 来源中明确描述的功能、实验结果、产品行为或事件 | Claude Code 有 Plan Mode；Magentic-One 有 Task Ledger 和 Progress Ledger；TravelPlanner 报告 GPT-4 success rate 为 0.6% |
| 观点 | 来源作者、厂商、社区成员或本文作者的判断 | Anthropic 建议优先使用简单 workflow；社区认为 plan mode 能减少失控改动 |
| 推断 | 本文从多个事实和观点中归纳出的工程含义 | Planning 应该成为 harness 层的 typed state，而不是只存在于模型隐藏推理中 |

## 一句话定义

比较稳妥的定义：

```text
Agent Planning 是把用户目标、环境状态、工具能力、约束条件和风险边界，
组织成可执行、可验证、可更新的任务状态，并在执行反馈中持续修正的工程活动。
```

它和相邻概念的边界：

| 概念 | 关注点 | 和 Planning 的关系 |
|---|---|---|
| Prompt engineering | 指令怎么表达 | 可以诱导模型写计划，但不负责状态和权限 |
| Reasoning | 模型如何推理 | 可以提高计划质量，但不等于可执行计划 |
| Context engineering | 每一步给模型看什么 | 给 planning 提供事实和状态 |
| Tool use | 模型能调用什么动作 | planning 决定何时、为何调用工具 |
| Agent loop | observe / think / act / observe 的循环 | planning 是 loop 中的控制策略之一 |
| Workflow | 预定义流程和分支 | planning 应优先落在 workflow 能表达的部分 |
| Harness engineering | 工具、权限、状态、验证、trace、handoff | planning 是 harness 的状态控制层 |

## 来源地图

| 范围 | 代表资料 | 本文使用方式 |
|---|---|---|
| 论文和技术报告 | PlanBench、NATURAL PLAN、TravelPlanner、LLMs Still Can't Plan、The Illusion of Thinking、LLM+P、PlanningBench、ReAct、Tree of Thoughts、Reflexion | 说明“规划能力”不是单轮问答能力，且需要状态、约束、验证和重规划 |
| Benchmark / eval | SWE-bench、WebArena、OSWorld、tau-bench、OpenAI Trace grading、Anthropic agent evals | 说明真实 agent planning 要看轨迹、工具调用、约束遵守和最终 artifact |
| 官方文档 | Claude Code Plan Mode、OpenAI Agent Builder / Agents SDK / trace grading、Google ADK workflow agents、GitHub Copilot coding agent、Google Jules | 观察厂商如何把 planning 做成权限、workflow、trace 和 human review |
| 开源项目 | LangGraph、AutoGen / Magentic-One、CrewAI、SWE-agent、OpenHands | 观察 planning object、state graph、ledger、Agent-Computer Interface 的实现形态 |
| 工程案例 | Claude Code、GitHub Copilot coding agent、Google Jules、Codex / Agents SDK | 说明 planning 正在和真实 repo、CI、PR、sandbox 绑定 |
| 开发者社区讨论 | Reddit / GitHub 社区里的 plan mode、background agent、LangGraph state 讨论 | 只作为痛点和使用方式线索，不作为事实主依据 |
| 事故复盘 | Replit 生产数据库删除事件、Air Canada chatbot 责任案、Knight Capital 自动交易事故 | 说明 planning 不能替代权限边界、rollback 和审计 |
| 招聘市场 | OpenAI Codex Agents 岗位、Anthropic agent eval / harness 文章、AI workflow / eval 岗位 | 说明市场能力要求从 prompt 转向 agent orchestration、eval、trace、runtime constraints |
| 历史类比 | STRIPS、PDDL、OR-Tools、DevOps pipeline、自动交易风控 | 帮助定位 LLM planning 的长期工程形态 |

## 最重要的结论

### 1. Planning 不是单点功能，而是一条控制链

**事实**：现代 agent 系统里，planning 通常同时出现在多个层面：

- Claude Code 把 Plan Mode 做成 read-only 权限模式，用于探索代码、解释方案和提出计划。
- OpenAI Agent Builder 把 workflow 表达成 agents、tools 和 control-flow logic，并支持 typed edges、preview run、trace grading。
- Microsoft AutoGen 的 Magentic-One 使用 Orchestrator 创建计划、维护 Task Ledger 和 Progress Ledger，并在停滞时修订计划。
- Google ADK 区分 LLM agent 和 deterministic workflow agents，例如 `SequentialAgent`、`ParallelAgent`、`LoopAgent`。
- CrewAI 的 `planning=True` 会在每次 crew iteration 前用 `AgentPlanner` 生成 step-by-step plan 并写回 task description。
- GitHub Copilot coding agent、Google Jules 这类异步 coding agent 都把“先计划、再批准、再执行、最后产出 PR / diff”做成产品交互。

**推断**：这些系统共同说明，Planning 不应该只被看成模型能力，而应被看成 harness 设计。一个 planning layer 至少要回答：

```text
当前目标是什么？
已经知道什么？
还需要查什么？
接下来允许做什么？
哪些动作必须审批？
计划执行到哪里了？
什么时候必须停下来重新规划？
```

### 2. 公开 benchmark 显示：自然语言 planning 远未解决

**事实**：不同 benchmark 正在测 agent planning 的不同侧面：

| Benchmark / Eval | 主要测什么 | 对 Planning 的启发 |
|---|---|---|
| [PlanBench](https://arxiv.org/abs/2206.10498) | 基于经典 planning domain 的动作、状态变化和计划生成 | 避免把常识检索误当规划能力；严格规划需要可验证状态 |
| [LLMs Still Can't Plan; Can LRMs?](https://arxiv.org/abs/2409.13373) | 用 PlanBench 初步评估 reasoning model 的 planning 能力 | reasoning model 进步明显，但仍不能等价于可靠 planner |
| [NATURAL PLAN](https://arxiv.org/abs/2406.04520) | Trip Planning、Meeting Planning、Calendar Scheduling | 自然语言场景也能系统化评测 plan 是否满足约束 |
| [TravelPlanner](https://arxiv.org/abs/2402.01622) | 带工具、数据和多约束的真实旅行规划 | 工具选择、约束跟踪和多目标权衡是主要失败点 |
| [PlanningBench](https://arxiv.org/abs/2605.20873) | 2026-05 提出的可扩展、可验证 planning 数据生成框架 | planning eval 正在从固定题库走向可生成、可验证数据 |
| [SWE-bench](https://arxiv.org/abs/2310.06770) | 真实 GitHub issue 修复 | 软件任务需要跨文件理解、测试反馈和多步执行 |
| [WebArena](https://arxiv.org/abs/2307.13854) | 真实网站上的长程 web task | 端到端成功率和人类差距暴露了环境交互难度 |
| [OSWorld](https://arxiv.org/abs/2404.07972) | 真实 OS 环境里的开放式计算机任务 | GUI / OS agent 需要 planning、感知、执行和恢复能力一起评估 |
| [tau-bench](https://arxiv.org/abs/2406.12045) | 动态用户、工具和 policy guideline 交互 | 真实业务 planning 还要处理用户变化和 policy 遵守 |

**事实**：TravelPlanner 摘要报告，即使 GPT-4 在该 benchmark 上也只有 0.6% success rate；论文指出 agent 容易偏离任务、工具使用不当或无法持续跟踪多重约束。

**事实**：WebArena 原始论文报告，最佳 GPT-4-based agent 的端到端任务成功率为 14.41%，明显低于 78.24% 的 human performance。

**事实**：SWE-bench 原始论文报告，早期最佳模型 Claude 2 只能解决 1.96% 的 issue，并指出这些任务需要理解和协调多函数、多类、多文件修改。

**推断**：不同 benchmark 的共同信号不是“模型没用”，而是：

```text
Planning 失败通常不是一句 plan 写错，而是状态、工具、约束、验证和恢复链条中某一环断了。
```

### 3. 厂商和框架正在把 Planning 外化成 workflow、ledger 和 trace

**事实**：[Anthropic Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) 区分 workflow 和 agent：workflow 走预定义代码路径，agent 自主决定流程和工具使用。Anthropic 建议优先选择足够简单、可组合的 workflow，只有任务真的需要自主决策时才用 agent。

**事实**：[Claude Code permission modes](https://code.claude.com/docs/en/permission-modes) 把 `plan` 描述为 reads-only，适合在修改前探索代码；批准计划后才切换到编辑或自动模式。

**事实**：[OpenAI Agent Builder](https://platform.openai.com/docs/guides/agent-builder) 把 workflow 定义为 agents、tools 和 control-flow logic 的组合，并支持 typed inputs / outputs、node、edge、preview runs 和 trace grading。[OpenAI Trace grading](https://platform.openai.com/docs/guides/trace-grading) 把 trace 定义为 agent 决策、工具调用和 reasoning steps 的端到端日志，并用于评估 correctness、quality 和 adherence。

**事实**：[Magentic-One](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/magentic-one.html) 的 Orchestrator 创建计划、维护 Task Ledger 和 Progress Ledger、委派 specialist agents，并动态修订计划。

**推断**：这些模式都在把 planning 从一次性文本变成运行时对象：

```text
plan item -> assigned actor -> allowed tools -> expected output -> verification -> progress ledger -> replan trigger
```

### 4. 计划越接近真实世界，越需要 deterministic checks

**事实**：经典 AI planning 从 [STRIPS](https://www.sciencedirect.com/science/article/pii/0004370271900105) 到 [PDDL](https://ipc08.icaps-conference.org/deterministic/PddlResources.html)，一直强调 state、actions、preconditions、effects 和 goal 的形式化表达。

**事实**：[LLM+P](https://arxiv.org/abs/2304.11477) 这类工作把 LLM 用作自然语言到 planning problem 的翻译层，再交给经典 planner 求解。

**事实**：[Google OR-Tools CP-SAT](https://developers.google.com/optimization/cp/) 把 constraint programming 描述为从巨大候选空间中识别可行解的技术，适用于 planning 和 scheduling。

**推断**：对 agent 工程来说，关键不是在“LLM planner”和“传统 planner”之间二选一，而是分工：

| 问题部分 | 更适合谁 |
|---|---|
| 读懂自然语言意图、识别上下文、提出候选方案 | LLM |
| 校验硬约束、求可行 schedule、证明 plan 是否满足 precondition | solver / deterministic checker |
| 决定高风险动作是否允许 | policy engine / approval gate |
| 在执行反馈后更新计划 | harness state machine + LLM |

### 5. 事故说明：Planning 不能替代权限边界

**事实**：2025-07，Replit AI agent 删除生产数据库事件被多家媒体和事件数据库记录。Replit CEO Amjad Masad 回应称该事件 unacceptable，并提到 automatic dev/prod DB separation、one-click restore，以及 planning-only / chat-only 方向的改进。参见 [TechTarget](https://www.techtarget.com/searchsoftwarequality/news/366627829/Replit-AI-agent-snafu-shot-across-the-bow-for-vibe-coding)、[Tom's Hardware](https://www.tomshardware.com/tech-industry/artificial-intelligence/ai-coding-platform-goes-rogue-during-code-freeze-and-deletes-entire-company-database-replit-ceo-apologizes-after-ai-engine-says-it-made-a-catastrophic-error-in-judgment-and-destroyed-all-production-data)、[OECD AI incident entry](https://oecd.ai/en/incidents/2025-07-19-1eb1)。

**事实**：2024-02，[Moffatt v. Air Canada, 2024 BCCRT 149](https://canlii.ca/t/k2spq) 中，Air Canada 因网站 chatbot 给出误导性 bereavement fare 信息而承担责任。这个案例更接近“错误事实 / 政策输出”而非 autonomous planning，但它说明企业不能把自动化系统说出的话外包给“模型自己负责”。

**事实**：2012-08-01，Knight Capital 自动交易系统事故造成重大市场影响。SEC 后续称 Knight 未能设置充分 safeguard 防止 erroneous orders，并以 1200 万美元和解。参见 [SEC 2013-222](https://www.sec.gov/newsroom/press-releases/2013-222)。

**推断**：这些事件的共同教训是：

```text
高风险系统不能只靠“计划正确”来保证安全。
需要让错误计划无法直接造成不可逆后果。
```

实际工程上，这意味着：

- read-only planning mode 和 write / execute mode 要分离。
- dev / staging / prod 资源要隔离。
- 删除、支付、发邮件、部署、改权限、触达生产数据等动作要有单独 approval。
- plan 需要被验证，但验证通过也不等于可绕过权限边界。
- rollback、backup、audit log 和 kill switch 是 planning layer 的下游必需品。

### 6. 招聘市场正在把 Planning 变成工程岗位能力

**事实**：2026-05 可查的 OpenAI [AI Systems Engineer, Codex Agents](https://openai.com/careers/ai-systems-engineer-codex-agents-san-francisco/) 岗位描述明确提到：agent harness、execution loop、tools、long-horizon tasks、orchestration、evals、logs / traces、runtime constraints 等能力。

**事实**：Anthropic 在 2026-01 发布 [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)，强调 agent eval 需要关注工具调用、reasoning、environment updates、trajectory 和 output grading。Anthropic 2026-03 的 [Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps) 也把 long-running coding agent harness 当作能力提升对象。

**事实**：非大厂岗位也开始把 LLM workflows、tool use、multi-step reasoning、agent orchestration、retries、safety checks、graceful degradation、evals 写进 JD，例如 Ashby 上的 [AI Engineer - Applied LLMs, Workflows & Evals](https://jobs.ashbyhq.com/delvo/0b8f4f50-c963-49f6-b2a6-d29a280d1ec4)。

**推断**：这说明市场正在从“会调 prompt”转向“会设计 agent execution system”。Planning 的能力要求更像平台工程、可靠性工程和 eval 工程的混合：

```text
agent orchestration + sandbox + trace + eval + policy + UX handoff
```

## Planning 的五层含义

### 第一层：人类战略规划

这一层回答：

```text
我们为什么要做这件事？
优先级是什么？
什么暂时不做？
成功后项目会变成什么样？
```

它对应 roadmap、strategy、product brief。它不是 agent runtime 的计划，但它是 agent planning 的上游事实来源。

本项目里已有相邻文档：[Coding Agent 计划文档分层](../../coding-agents/research/workflows/03-plan-documents-for-coding-agents.md)。那篇文档关注 Roadmap、Spec、Execution Plan 和 Progress Status 的文档分层。本文关注的是 agent runtime 如何使用 planning。

### 第二层：任务规划

这一层回答：

```text
给定目标后，任务应该拆成哪些子任务？
哪些子任务可以并行？
哪些子任务必须先完成？
哪个 actor 负责哪个子任务？
```

典型系统：

- Magentic-One orchestrator 规划并委派。
- OpenAI handoffs 把任务交给 specialist agent。
- CrewAI `AgentPlanner` 给每个 task 添加 step-by-step plan。
- LangGraph plan-and-execute agent 先规划，再逐步执行，并在必要时 re-plan。

### 第三层：执行规划

这一层回答：

```text
下一步具体调用哪个工具？
调用前需要哪些参数？
调用后如何判断成功？
失败时重试、换工具、回退还是请求人类？
```

ReAct、Claude think tool、OpenAI agent loop、Claude Code Plan Mode 都属于这一层附近。这里的重点不是 plan 文本漂亮，而是动作和观察之间能闭环：

```text
Thought / Plan -> Action -> Observation -> State Update -> Next Action
```

### 第四层：运行时进度跟踪和重规划

这一层回答：

```text
计划执行到哪里了？
哪些假设已被证伪？
当前是不是卡住了？
是否应该重排步骤、缩小目标或请求人类？
```

Magentic-One 的 Progress Ledger、LangGraph 的 state graph、OpenAI trace、Anthropic evals 都在强调这件事：长期任务不能只靠上下文窗口里的自然语言记忆。

**推断**：一个可靠的 long-horizon agent 应该有显式 progress state，例如：

```yaml
goal: "add Google OAuth"
status: "in_progress"
current_step: "write callback route tests"
completed:
  - "inspect auth/session flow"
  - "identify env var conventions"
blocked: []
assumptions:
  - "existing session cookie abstraction can support OAuth"
next_actions:
  - tool: "edit"
    target: "auth callback tests"
    risk: "low"
verification:
  - "npm test -- auth"
approval_required:
  - "change production auth config"
```

### 第五层：约束规划和资源调度

这一层回答：

```text
有没有硬约束？
是否需要最优或近似最优 schedule？
是否存在资源容量、时间窗口、依赖关系、合规限制？
```

这不是 LLM 最擅长的部分。旅行规划、会议安排、工厂排程、供应链、CI 并发、部署窗口都更接近 constraint planning / scheduling。

**推断**：当约束可形式化时，不要让模型凭自然语言“记住”。应把 LLM 放在解释器和协调器位置，把可验证约束交给 solver、rule engine 或静态 checker。

## 论文和技术报告脉络

### 经典 AI Planning：规划从来不只是“写步骤”

**事实**：General Problem Solver、STRIPS、PDDL 这条线把 planning 定义为在状态空间中寻找动作序列，使初始状态通过满足 preconditions 的 actions 到达 goal state。

关键概念：

- state：世界当前是什么样。
- action / operator：可以做什么。
- precondition：动作执行前必须满足什么。
- effect：动作执行后世界如何改变。
- goal：什么状态算成功。
- search：如何在巨大动作空间中找路径。

**推断**：LLM agent 常见失败可以用这套词重新描述：

| LLM agent 失败 | 经典 planning 视角 |
|---|---|
| 忘记用户约束 | goal / constraint 没有显式编码 |
| 执行不存在的动作 | action schema 不完整或工具选择错误 |
| 在错误前提下行动 | precondition 没有检查 |
| 做完后没有验证 | effect 没有观测或校验 |
| 卡住后反复尝试 | search / replanning 策略缺失 |

### Prompt planning：从 CoT 到 ReAct、ToT、Reflexion

**事实**：[ReAct](https://arxiv.org/abs/2210.03629) 把 reasoning traces 和 task-specific actions 交错生成，使模型能根据 observation 更新 action plan。

**事实**：[Plan-and-Solve Prompting](https://arxiv.org/abs/2305.04091) 针对 multi-step reasoning，要求模型先制定计划再解决。

**事实**：[Tree of Thoughts](https://arxiv.org/abs/2305.10601) 把中间 thought 看成可搜索、可评估、可回溯的候选路径。

**事实**：[Reflexion](https://arxiv.org/abs/2303.11366) 用 verbal feedback / episodic memory 改善后续尝试。

**观点**：这些工作推动了“让模型显式规划”的实践，但它们大多仍在 prompt / inference 策略层面。

**推断**：对工程系统来说，它们的真正价值不是某个 prompt 模板，而是三条原则：

1. 不要把所有决策压进一次模型输出。
2. 允许在执行反馈后更新计划。
3. 让候选路径、失败原因和验证信号成为外部可检查对象。

### Reasoning model 不是 planning system

**事实**：Apple 的 [The Illusion of Thinking](https://machinelearning.apple.com/research/illusion-of-thinking) 技术报告用可控 puzzle environment 分析 reasoning models，并指出模型在复杂度提升后可能出现 accuracy collapse 和 reasoning effort decline。

**事实**：[LLMs Still Can't Plan; Can LRMs?](https://arxiv.org/abs/2409.13373) 用 PlanBench 初步评估 OpenAI o1，结论是 reasoning model 相比旧模型有明显进步，但仍没有饱和严格 planning benchmark。

**观点**：这些论文在社区中有争议，尤其是 benchmark 是否公平、是否把 planning 限定得过窄、是否低估了工具使用和外部 verifier 的价值。

**推断**：比较稳妥的工程结论是：

```text
Reasoning model 可以提高 plan 质量，但不能替代 plan representation、verification、permissions 和 rollback。
```

## 官方文档和框架实践

### Anthropic：Plan Mode 和 workflow 优先

**事实**：Claude Code 的 permission modes 文档把 `plan` 模式描述为 reads-only，适合在修改前探索代码、运行只读分析和提出计划。批准计划后，用户可以选择进入 auto、accept edits 或逐项 review 编辑。

**事实**：Claude Code best practices 推荐 `Explore -> Plan -> Implement -> Commit`，并建议在不确定实现方式、需要多文件修改或不熟悉代码库时先计划。

**事实**：Anthropic Building Effective Agents 强调，能用 workflow 清楚表达的任务，不要过早升级成复杂 autonomous agent。agent 适合 open-ended、需要环境反馈和多步工具使用的任务。

**事实**：Anthropic 的 ["think" tool](https://www.anthropic.com/engineering/claude-think-tool) 把“停下来思考”作为工具调用过程中的显式空间，尤其适合复杂工具使用、policy-heavy environments 和 sequential decisions。

**推断**：Anthropic 的 planning 观可以概括为：

```text
先读上下文，再写计划；先用 workflow 固定结构，再用 agent 处理不确定性；
把思考空间放到工具循环中，而不是只放在开头。
```

### OpenAI：workflow、handoff、guardrail、trace

**事实**：OpenAI Agent Builder 把 workflow 定义为 agents、tools 和 control-flow logic 的组合；节点之间是 typed edge，开发者可以预览运行并导出 SDK code。

**事实**：OpenAI trace grading 把 trace 视为 agent 决策、工具调用、reasoning steps 的端到端日志，用来打分、定位错误和做 eval。

**事实**：OpenAI Agents SDK / Agent Builder 体系把 agent workflow 的工程对象拆成 agent、tool、handoff、guardrail、session、trace 等模块。

**推断**：OpenAI 的 planning 观可以概括为：

```text
把 agent plan 放进 workflow，并用 handoff、guardrail、trace 和 eval 包住它。
```

### LangGraph：Plan-and-Execute 和 state graph

**事实**：LangChain 的 [Plan-and-Execute Agents](https://www.langchain.com/blog/plan-and-execute-agents) 介绍了一类先规划、后执行、再 replanning 的 agent 架构。

**事实**：LangGraph 让 agent workflow 表达为 stateful graph，这让计划、执行、重试、分支和停止条件更容易显式建模。

**推断**：LangGraph 的价值不在“让模型更聪明”，而在让开发者控制：

- 哪些步骤由 LLM 决策。
- 哪些步骤由代码固定。
- 失败后回到哪个节点。
- 中间状态如何持久化。

### AutoGen / Magentic-One：ledger 是 planning 的核心对象

**事实**：Magentic-One 的 Orchestrator 创建计划、维护 Task Ledger / Progress Ledger、委派 specialist agent，并动态修订计划。

**推断**：Ledger 的关键价值是把“计划”和“执行事实”分离：

```text
Task Ledger: 目标、团队能力、总体计划
Progress Ledger: 当前进展、已完成事项、卡点、下一步
```

这比让模型在长上下文里自己记住进度更可靠。

### Google ADK：确定性 workflow agent 和 LLM agent 分层

**事实**：Google ADK 文档区分 `LlmAgent` 和 deterministic workflow agents。`LoopAgent` 被描述为非 LLM 驱动的 deterministic controller，会循环执行 sub-agents 直到满足 termination condition。

**推断**：这是一条重要工程原则：

```text
流程控制尽量用确定性代码；
不确定的理解、生成、判断再交给 LLM。
```

### CrewAI：Planning 作为 crew iteration 前置步骤

**事实**：CrewAI docs 的 planning feature 会在每次 Crew iteration 前，把 crew 信息发给 `AgentPlanner`，生成 step-by-step plan 并追加到每个 task description。

**推断**：这代表一种轻量做法：不一定要先做复杂 state machine，也可以把 planning 作为任务增强层。但风险是 plan 仍可能只是自然语言，缺少独立 verifier 和权限边界。

### SWE-agent / OpenHands：ACI 比 plan 文本更重要

**事实**：[SWE-agent](https://arxiv.org/abs/2405.15793) 论文提出 Agent-Computer Interface，认为语言模型 agent 需要专门设计的 computer interface。SWE-agent 的 custom ACI 改善了代码导航、编辑、测试执行能力。

**事实**：[OpenHands](https://github.com/All-Hands-AI/OpenHands) 这类开源项目把 coding agent 放进可执行环境，让它能读写代码、运行命令、通过 issue / PR 工作流交付结果。

**推断**：软件工程 planning 很大程度取决于 ACI。一个模型即使能写出合理计划，如果搜索文件、编辑文件、运行测试、观察 diff 的接口很差，最终仍然会失败。

## 工程案例

### Claude Code：计划模式作为权限和认知边界

**事实**：Claude Code 的 Plan Mode 是只读分析和计划模式，适合探索复杂代码库和多文件变更。

**推断**：Plan Mode 的核心不是“输出计划”，而是“在行动前禁止行动”。

```text
Planning mode 应该首先是权限模式，然后才是 UI 模式。
```

也就是说，用户点击“计划”时，系统应从权限上阻止写文件、删数据、发请求、部署，而不是只要求模型“别动”。

### GitHub Copilot coding agent：issue 到 PR 的异步 planning

**事实**：GitHub 文档描述 Copilot coding agent 会在 GitHub Actions 支持的 ephemeral development environment 中工作，可以探索代码、修改、运行 tests / linters，并通过 draft pull request workflow 交付。

**事实**：GitHub 文档还说明 Copilot coding agent 在受限开发环境中运行，网络访问由 firewall 控制；其提出的 draft PR 需要有写权限的人批准后才能运行 Actions workflows。

**推断**：GitHub 的设计把 Planning 放在 issue / PR / CI 环境中，而不是聊天窗口中：

- issue 描述提供任务目标。
- branch / PR 提供变更边界。
- Actions 环境提供隔离执行。
- tests / linters 提供反馈。
- human review 决定合并。

### Google Jules：先计划，后在 Cloud VM 里执行

**事实**：Google Jules 官方页面描述，Jules 会 fetch repository、clone 到 Cloud VM、develop a plan，并验证变更。Jules docs 描述，提交任务后 Jules 会生成计划，用户可以在任何代码修改前 review and approve。

**推断**：异步 coding agent 的用户体验正在收敛：

```text
给任务 -> agent 读仓库 -> 产出计划 -> 用户批准 -> agent 执行 -> PR / diff / 验证结果 -> 人类 review
```

### OpenAI Codex / Agents SDK：Planning 和 trace 进入平台层

**事实**：OpenAI Codex Agents 岗位描述把 long-horizon tasks、agent harness、execution loop、orchestration、evals、logs / traces、runtime constraints 写进岗位职责。

**推断**：Codex 类系统的竞争点不是“是否会生成计划”，而是：

- 能否把计划映射到真实开发环境。
- 能否在工具调用中持续更新状态。
- 能否用 trace 解释为什么这样做。
- 能否在失败样本上做 eval 和 harness 改进。

## 开发者社区讨论

社区讨论不是事实主依据，但能暴露真实使用痛点。

### Plan Mode 被当成防失控工具

**观点**：Claude Code / GitHub Copilot / LangGraph / AutoGPT / OpenHands 相关社区里，开发者普遍把 plan 文件、plan mode、slash command、spec 文档用作防止 agent 漫游的办法。

典型做法：

- 先让 agent 只读代码并写计划。
- 人类编辑计划。
- 再让 agent 按计划执行。
- 执行后回到 plan / spec 对照 diff。

**推断**：这说明“计划”在开发者心中不是形式主义，而是 review surface。它让人类能在代码改动前审查 agent 的意图。

### 社区痛点：计划会漂移、重复、忽略或过度工程

**观点**：社区讨论中常见抱怨包括：

- agent 写了计划但执行时偏离计划。
- plan mode 反复提出计划，不进入执行。
- agent 忽略某些 plan / slash command。
- 计划过大，导致一次 diff 难 review。
- plan 里没有测试和验收标准。
- 背景 agent 消耗请求或费用，用户不知道它在做什么。

**推断**：只生成计划文本不够。需要：

- plan item ID。
- 每个 item 的 acceptance check。
- 执行时把 diff / tool call 归因到 plan item。
- 偏离计划时暂停或重新请求 approval。

### AutoGPT 早期热潮的教训

**事实**：AutoGPT 早期把“给一个目标后自主循环拆任务”推到公众视野，但社区很快发现泛化 autonomous loop 容易 hallucinate、迷路、消耗 token、缺乏可靠验证。

**观点**：社区后来逐渐转向更具体的框架：coding agent、browser agent、workflow agent、domain agent、PR-based agent。

**推断**：这是一条重要历史线：

```text
通用自主规划的 demo 很容易做；
可靠领域规划的 harness 很难做。
```

## 事故和失败模式

### 失败模式 1：计划权限错配

症状：

- 用户以为 agent 在 planning / chat-only。
- agent 实际拥有写入、删除或生产访问权限。
- 模型遵循意图失败时，系统没有硬边界。

代表案例：

- Replit 删除生产数据库事件。

工程对策：

- planning mode 必须是 read-only permission mode。
- prod credential 不应出现在 agent 默认环境。
- destructive tool 默认不可见，或强制 approval。
- agent 不能自己提升权限。

### 失败模式 2：计划约束丢失

症状：

- 多个约束在长上下文中被遗忘。
- 计划满足显性需求，但违反隐性常识或政策。
- 工具返回信息后没有更新约束集。

代表 benchmark：

- TravelPlanner。
- NATURAL PLAN。
- tau-bench。

工程对策：

- 把约束提取成结构化 checklist。
- 每个 plan item 执行前检查相关约束。
- 高约束场景使用 solver / rule engine / validator。

### 失败模式 3：计划和进度混在一起

症状：

- 计划里充满执行日志。
- agent 不知道哪些步骤已完成。
- 新会话接手时无法判断真实状态。

工程对策：

- 分离 `Plan` 和 `Progress Status`。
- 使用 ledger 记录 completed / current / blocked / next。
- 每轮工具执行后更新状态。

### 失败模式 4：计划无法验证

症状：

- 计划只有“实现功能”“优化代码”这类模糊步骤。
- 没有测试、lint、snapshot、用户验收或人工 review。
- agent 宣称完成，但没有外部证据。

工程对策：

- 每个计划项必须有 verification。
- trace 中保留工具调用、命令输出、diff 和错误。
- eval 从最终结果扩展到 trace grading。

### 失败模式 5：计划过度自由

症状：

- agent 自己决定是否重构、是否升级依赖、是否改架构。
- diff 变大，review 成本超过人工实现。
- 小任务被扩张成大改动。

工程对策：

- 在 plan 中显式写 non-goals。
- 为每个 task 设置 edit budget / file boundary。
- 偏离计划必须请求人类确认。
- 小任务不进入重型 planning。

## Historical Analogies

### 类比 1：经典 planner 和 LLM planner

经典 planner 强在：

- 状态显式。
- 约束可验证。
- 搜索可解释。
- 成功条件明确。

LLM planner 强在：

- 理解模糊自然语言。
- 总结上下文。
- 生成候选方案。
- 在不完整信息下提出下一步。

**推断**：最可靠的 agent planning 不是让 LLM 取代 planner，而是让 LLM 负责“建模和协调”，让 planner / checker / tests 负责“验证和约束”。

### 类比 2：自动交易和自动 agent

Knight Capital 事件说明：自动化系统的危险不只是“算法错”，还包括部署、监控、kill switch、风险限额、环境隔离和操作流程失败。

**推断**：Agent planning 也会走同样路径。真正的问题不是模型有没有计划，而是：

```text
当错误计划开始执行时，系统有没有办法及时限制损失？
```

### 类比 3：DevOps pipeline 和 agent workflow

成熟 DevOps 不会让人直接在 prod 上手动执行所有步骤，而是通过 pipeline、review、staging、rollback 和 audit 控制风险。

**推断**：Agent workflow 也应该类似：

```text
plan -> review -> isolated execution -> tests -> artifact -> review -> merge/deploy
```

这比“聊天窗口里让模型直接改一切”更接近可持续工程。

### 类比 4：行程规划和软件规划

TravelPlanner 这类 benchmark 看似离 coding agent 很远，但它暴露的问题很像软件工程：

- 多约束。
- 多工具。
- 长链条。
- 用户偏好不完整。
- 中途信息可能改变。
- 最终计划必须整体一致。

**推断**：Coding agent 规划功能不能只用 toy task 验证。它需要处理“局部步骤都合理，但整体方案不成立”的失败。

## Planning Harness 设计框架

下面是可迁移到 agent runtime 的 planning harness 框架。

### 1. Plan Object

不要只保存自然语言计划。至少保存：

```yaml
id: "plan-2026-05-30-oauth"
goal: "Add Google OAuth login"
non_goals:
  - "Do not migrate session storage"
scope:
  files_may_touch:
    - "src/auth/**"
    - "tests/auth/**"
  files_must_not_touch:
    - "infra/prod/**"
risk_level: "medium"
steps:
  - id: "P1"
    description: "Inspect existing session and env var flow"
    mode: "read-only"
    allowed_tools: ["read", "search"]
    verification: "summarize relevant files with paths"
  - id: "P2"
    description: "Add callback route tests"
    mode: "write"
    allowed_tools: ["edit", "test"]
    verification: "auth tests fail for missing callback"
approval_required:
  - "adding new OAuth provider secrets"
replan_triggers:
  - "test failure unrelated to changed files"
  - "need to touch files outside scope"
  - "missing dependency"
```

### 2. Context Acquisition Gate

Plan 之前先规定 agent 必须收集哪些上下文：

- 项目入口 README / AGENTS。
- 相关模块文件。
- 测试和 CI。
- 近期错误日志。
- 已有 issue / PR / docs。
- 相关外部官方文档。

适合写成 checklist，而不是笼统说“理解项目”。

### 3. Plan Review Gate

执行前 review 不只看计划是否合理，还要看：

- 是否有 non-goals。
- 是否限制了 scope。
- 是否包含 verification。
- 是否有 rollback / abort 条件。
- 是否声明风险。
- 是否区分 read-only、write、execute、destructive。

### 4. Tool Boundary

计划项应该绑定工具权限：

| Step 类型 | 默认权限 |
|---|---|
| Explore | read / search |
| Design | read / search / no mutation |
| Implement | edit / patch / local test |
| Validate | test / lint / build |
| External action | approval required |
| Destructive action | deny by default |
| Production action | separate workflow |

### 5. Progress Ledger

每次执行后写入：

```yaml
completed:
  - step: "P1"
    evidence: "summarized auth/session.ts and env.ts"
current: "P2"
blocked: []
observations:
  - "existing test helper can create session cookie"
changes:
  - "tests/auth/oauth-callback.test.ts"
next: "run auth test suite"
```

### 6. Replan Policy

需要明确什么时候不能继续原计划：

- 工具输出和 plan 假设冲突。
- 需要访问计划外文件。
- 测试失败原因超出当前 step。
- 用户需求变更。
- 成本或时间超过预算。
- 发现安全、合规、隐私风险。

重规划不等于从头开始。好的 replan 应只更新受影响 steps，并保留已验证事实。

### 7. Trace 和 Eval

Planning eval 不应该只看最终答案。至少要看：

- 是否收集了必要上下文。
- 是否识别关键约束。
- 是否正确选择工具。
- 是否在高风险动作前请求审批。
- 是否按计划执行。
- 是否发现并处理失败。
- 是否产出可验证 artifact。

OpenAI trace grading 和 Anthropic agent evals 都在往这个方向走。

## 什么时候应该使用 Planning

适合使用 Planning：

- 修改多个文件或多个模块。
- 任务目标不清楚，需要先探索。
- 需要外部文档、业务规则或政策。
- 有安全、权限、合规、生产数据风险。
- 需要长时间执行或异步执行。
- 需要多 agent / subagent 协作。
- 需要人类 review 后再执行。
- 失败成本高，或回滚困难。

不需要重型 Planning：

- 修 typo。
- 改单行配置。
- 明确的小 bug，有直接复现和测试。
- 已有 deterministic script 能完成。
- 人类已经给出精确 patch。

一个简单规则：

```text
如果错误执行的成本高于写计划的成本，就先 planning。
如果写计划的成本高于任务本身，就直接执行并验证。
```

## 对个人 coding agent 工作流的建议

面向本仓库的长期研究和写作工作，可以采用以下约定：

1. 文档调研类任务：先写 source map，再写 outline，再写正文。
2. 多文件工程任务：先创建 `docs/plans/<date>-<topic>.md` 或临时 plan，再执行。
3. 小改动：直接执行，但 final answer 必须说明验证方式。
4. 高风险动作：要求 plan 中显式标出 risk、approval、rollback。
5. 长任务：维护 progress status，而不是依赖会话上下文。
6. agent 失败后：不要只改 prompt，优先问缺少哪类 planning harness。

推荐最小模板：

```md
# Plan: 任务名

日期：YYYY-MM-DD

## Goal

## Non-goals

## Context Read

## Steps

| ID | Step | Files / Tools | Verification | Risk |
|---|---|---|---|---|
| P1 |  |  |  |  |

## Replan Triggers

## Current Status
```

## 对 harness engineering 的启发

Planning 在 harness engineering 中不是一个装饰层，而是把模型自由度变成工程可控性的关键接口。

一个成熟 harness 应该让以下问题都有外部答案：

- agent 为什么认为这是目标？
- 它基于哪些上下文制定计划？
- 它准备调用哪些工具？
- 哪些动作需要人类批准？
- 它执行到了哪一步？
- 哪一步失败了？
- 它为什么重规划？
- 最终结果通过了什么验证？
- 人类能否回放、审计和改进这次 run？

最终判断：

> Planning 的下一步不是更长的计划文本，而是更好的计划对象、状态账本、工具边界、验证器和 trace。

## 来源

### 论文和技术报告

- [PlanBench: An Extensible Benchmark for Evaluating Large Language Models on Planning and Reasoning about Change](https://arxiv.org/abs/2206.10498)
- [LLMs Still Can't Plan; Can LRMs? A Preliminary Evaluation of OpenAI's o1 on PlanBench](https://arxiv.org/abs/2409.13373)
- [NATURAL PLAN: Benchmarking LLMs on Natural Language Planning](https://arxiv.org/abs/2406.04520)
- [TravelPlanner: A Benchmark for Real-World Planning with Language Agents](https://arxiv.org/abs/2402.01622)
- [PlanningBench: Generating Scalable and Verifiable Planning Data for Evaluating and Training Large Language Models](https://arxiv.org/abs/2605.20873)
- [SWE-bench: Can Language Models Resolve Real-World GitHub Issues?](https://arxiv.org/abs/2310.06770)
- [WebArena: A Realistic Web Environment for Building Autonomous Agents](https://arxiv.org/abs/2307.13854)
- [OSWorld: Benchmarking Multimodal Agents for Open-Ended Tasks in Real Computer Environments](https://arxiv.org/abs/2404.07972)
- [tau-bench: A Benchmark for Tool-Agent-User Interaction in Real-World Domains](https://arxiv.org/abs/2406.12045)
- [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)
- [Plan-and-Solve Prompting](https://arxiv.org/abs/2305.04091)
- [Tree of Thoughts](https://arxiv.org/abs/2305.10601)
- [Reflexion: Language Agents with Verbal Reinforcement Learning](https://arxiv.org/abs/2303.11366)
- [LLM+P: Empowering Large Language Models with Optimal Planning Proficiency](https://arxiv.org/abs/2304.11477)
- [The Illusion of Thinking](https://machinelearning.apple.com/research/illusion-of-thinking)
- [SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering](https://arxiv.org/abs/2405.15793)

### 官方文档、博客和框架

- [Anthropic: Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents)
- [Anthropic: Best practices for Claude Code](https://code.claude.com/docs/en/best-practices)
- [Anthropic: Claude Code permission modes](https://code.claude.com/docs/en/permission-modes)
- [Anthropic: The "think" tool](https://www.anthropic.com/engineering/claude-think-tool)
- [Anthropic: Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- [Anthropic: Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps)
- [OpenAI Agent Builder](https://platform.openai.com/docs/guides/agent-builder)
- [OpenAI Trace grading](https://platform.openai.com/docs/guides/trace-grading)
- [OpenAI Agents SDK guardrails](https://openai.github.io/openai-agents-js/guides/guardrails)
- [OpenAI: A practical guide to building agents](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf)
- [LangChain: Plan-and-Execute Agents](https://www.langchain.com/blog/plan-and-execute-agents)
- [AutoGen: Magentic-One](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/magentic-one.html)
- [Google ADK workflow agents](https://google.github.io/adk-docs/agents/workflow-agents/)
- [Google ADK LoopAgent](https://google.github.io/adk-docs/agents/workflow-agents/loop-agents/)
- [CrewAI Planning](https://docs.crewai.com/en/concepts/planning)
- [GitHub Copilot coding agent](https://docs.github.com/en/copilot/using-github-copilot/coding-agent/about-assigning-tasks-to-copilot)
- [GitHub Copilot cloud agent responsible use](https://docs.github.com/en/enterprise-cloud@latest/copilot/responsible-use/copilot-cloud-agent)
- [Google Jules](https://jules.google/)
- [Google Jules docs](https://jules.google/docs/)

### 历史和类比

- [STRIPS: A new approach to the application of theorem proving to problem solving](https://www.sciencedirect.com/science/article/pii/0004370271900105)
- [PDDL resources, IPC 2008](https://ipc08.icaps-conference.org/deterministic/PddlResources.html)
- [Planning Domain Definition Language overview](https://planning.wiki/guide/whatis/pddl)
- [Google OR-Tools Constraint Optimization](https://developers.google.com/optimization/cp/)
- [NASA Planning and Scheduling Group](https://www.nasa.gov/intelligent-systems-division/autonomous-systems-and-robotics/planning-and-scheduling-group/)
- [SEC: Knight Capital Market Access Rule action](https://www.sec.gov/newsroom/press-releases/2013-222)

### 事故和社区讨论

- [TechTarget: Replit AI agent snafu](https://www.techtarget.com/searchsoftwarequality/news/366627829/Replit-AI-agent-snafu-shot-across-the-bow-for-vibe-coding)
- [Tom's Hardware: Replit AI coding platform incident](https://www.tomshardware.com/tech-industry/artificial-intelligence/ai-coding-platform-goes-rogue-during-code-freeze-and-deletes-entire-company-database-replit-ceo-apologizes-after-ai-engine-says-it-made-a-catastrophic-error-in-judgment-and-destroyed-all-production-data)
- [OECD AI incident entry: Replit AI coding tool deletes live production database](https://oecd.ai/en/incidents/2025-07-19-1eb1)
- [Moffatt v. Air Canada, 2024 BCCRT 149](https://canlii.ca/t/k2spq)
- [Reddit: plan-only mode but writing plans to files](https://www.reddit.com/r/ClaudeCode/comments/1mqfq4y/planonly_mode_but_writing_the_plans_to_files/)
- [Reddit: LangGraph observe / act / verify / replan pattern](https://www.reddit.com/r/LangChain/comments/1r0ov7l/a_simple_pattern_for_langgraph_observe_act_verify/)
- [Reddit: production agents and workspace state](https://www.reddit.com/r/LangChain/comments/1tafflp/for_production_agents_im_starting_to_think/)

### 招聘市场

- [OpenAI Careers: AI Systems Engineer, Codex Agents](https://openai.com/careers/ai-systems-engineer-codex-agents-san-francisco/)
- [Anthropic Engineering](https://www.anthropic.com/engineering)
- [AI Engineer - Applied LLMs, Workflows & Evals](https://jobs.ashbyhq.com/delvo/0b8f4f50-c963-49f6-b2a6-d29a280d1ec4)
