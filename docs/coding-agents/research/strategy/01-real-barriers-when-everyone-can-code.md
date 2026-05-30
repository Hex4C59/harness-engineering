# 人人皆可 Coding 的时代，真正的壁垒是什么

调研日期：2026-05-30

## 这篇文档回答什么

这个问题不是在问“AI 会不会写代码”，而是在问：

```text
当写代码这件事越来越便宜，什么能力仍然稀缺？
什么东西不会因为模型代码能力提升而自动被抹平？
```

核心结论：

> 人人皆可 coding 之后，壁垒不会消失，而是从“把想法翻译成代码”迁移到“把真实世界的问题翻译成可验证、可维护、可运营的软件系统”。

换句话说，代码生成能力变成公共基础设施后，真正拉开差距的是：

- 能不能定义正确问题。
- 能不能提供高质量上下文。
- 能不能设计可执行验证。
- 能不能把 agent 放进可靠 harness。
- 能不能维护长期架构、数据、权限、安全和运营。
- 能不能获得用户信任和分发。

这和本仓库的主题是一致的：未来的软件工程壁垒，很大一部分会变成 **harness engineering** 壁垒。

## 来源说明

本文优先使用官方文档、论文、技术报告、benchmark、开源项目和一线工程经验。社区讨论只作为观察入口，不单独作为事实依据。

| 来源 | 类型 | 主要价值 |
|---|---|---|
| [AI Harness Engineering](https://arxiv.org/abs/2605.13357) | 论文 | 把软件工程 agent 能力解释为 model-harness-environment 系统，而不是裸模型能力 |
| [SWE-bench](https://openreview.net/forum?id=VTF8yNQM66) / [官网](https://www.swebench.com/original.html) | Benchmark / 论文 | 用真实 GitHub issue + 测试验证衡量 agent 是否能修真实仓库问题 |
| [SWE-agent](https://arxiv.org/abs/2405.15793) / [GitHub](https://github.com/SWE-agent/SWE-agent) | 开源项目 / 论文 | 说明 agent-computer interface、编辑器、搜索、测试接口会显著影响结果 |
| [Terminal-Bench](https://arxiv.org/abs/2601.11868) / [官网](https://www.tbench.ai/) | Benchmark | 评估 agent 在真实终端环境里完成多步骤任务的能力 |
| [LiveCodeBench](https://arxiv.org/abs/2403.07974) / [GitHub](https://github.com/LiveCodeBench/LiveCodeBench) | Benchmark | 强调动态更新、污染控制、自修复、执行和测试输出预测 |
| [METR long task measurement](https://metr.org/blog/2025-03-19-measuring-ai-ability-to-complete-long-tasks/) | 技术报告 | 用“任务时间跨度”衡量 agent 能可靠完成多长任务 |
| [METR experienced OSS developer productivity study](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/) | RCT / 技术报告 | 早 2025 AI 工具让熟悉大型仓库的资深开源开发者变慢，提醒不能只看 demo |
| [METR uplift update](https://metr.org/blog/2026-02-24-uplift-update/) | 技术报告更新 | 说明 AI productivity 结论会随工具快速演化，需要持续复测 |
| [GitHub Copilot productivity research](https://github.blog/news-insights/research/research-quantifying-github-copilots-impact-on-developer-productivity-and-happiness/) / [论文](https://arxiv.org/abs/2302.06590) | 实验 / 官方研究 | 在受控 JS HTTP server 任务中，Copilot 组完成更快 |
| [DORA 2025 State of AI-assisted Software Development](https://dora.dev/dora-report-2025) | 行业报告 | AI 是组织能力放大器，会放大强项，也会放大弱项 |
| [Stack Overflow 2025 Developer Survey: AI](https://survey.stackoverflow.co/2025/ai) | 开发者调查 | AI 使用上升，但开发者对准确性的信任不足，验证仍是核心问题 |
| [Codex best practices](https://developers.openai.com/codex/learn/best-practices) | 官方文档 | 强调 task context、`AGENTS.md`、测试、review、MCP、skills、automation |
| [Codex AGENTS.md](https://developers.openai.com/codex/guides/agents-md) | 官方文档 | 项目指令分层、可复用上下文和仓库规范的官方机制 |
| [Codex Skills](https://developers.openai.com/codex/skills) | 官方文档 | 把重复工作流封装成可发现、渐进加载的技能 |
| [Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop) | 官方技术博客 | 解释 Codex agent loop 如何组织模型、上下文、工具和终止状态 |
| [Unlocking the Codex harness](https://openai.com/index/unlocking-the-codex-harness) | 官方技术博客 | Codex harness 不只是模型调用，还包括 thread、配置、认证、sandbox、MCP、skills |
| [Running Codex safely](https://openai.com/index/running-codex-safely) | 官方安全实践 | sandbox、approval、network policy、agent-native telemetry 是规模化使用的安全边界 |
| [OpenAI Codex Core Agent 岗位](https://openai.com/careers/applied-ai-engineer-codex-core-agent-san-francisco/) | 招聘市场 | 岗位要求明确提到 eval、failure modes、tool-use、context construction 和 agent robustness |
| [Claude Code best practices](https://www.anthropic.com/engineering/claude-code-best-practices) | 官方工程博客 | 强调计划、上下文、工具权限、测试、review 和安全使用 |
| [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) | 官方工程博客 | 长任务需要 progress file、git history、环境初始化和分阶段执行 |
| [Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) | 官方工程博客 | 技能通过渐进式上下文加载包装组织知识和流程 |
| [Anthropic postmortem](https://www.anthropic.com/engineering/a-postmortem-of-three-recent-issues/) | 事故复盘 | 模型/产品质量下降也需要检测、回滚和防复发测试 |
| [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications) | 安全框架 | Prompt injection、supply chain、excessive agency 等风险成为 agent 工程约束 |
| [Invariant Labs MCP tool poisoning](https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks) | 安全研究 | MCP tool metadata 本身可成为攻击面 |
| [CVE-2025-54136](https://nvd.nist.gov/vuln/detail/CVE-2025-54136) | 漏洞数据库 | Cursor MCP 信任机制漏洞说明 agent 工具配置也需要治理 |
| [Low-code/no-code adoption SLR](https://www.sciencedirect.com/science/article/pii/S0164121224003443) | 历史类比 / 综述 | 低门槛开发会带来治理、架构、安全、协作和可扩展性问题 |
| [End-user development mapping study](https://www.sciencedirect.com/science/article/pii/S0164121218302577) | 历史类比 / 综述 | 非专业开发者创造软件不是新现象，质量和维护一直是核心难题 |

## 事实、观点和推断

这类题很容易被写成口号，所以先分清三层：

| 类型 | 本文用法 |
|---|---|
| 事实 | 来源中明确出现的数据、机制、产品能力、岗位要求、漏洞或 benchmark 设计 |
| 观点 | 作者、工程团队、社区对这些事实的解释和经验总结 |
| 推断 | 基于多类来源共同指向的趋势，整理成本文的判断框架 |

本文的结论不是说“写代码不重要了”。更准确地说：

```text
写代码仍然重要，但它不再是唯一稀缺环节。
稀缺点正在上移到定义、验证、集成、治理和反馈闭环。
```

## 事实一：代码生成能力已经从“稀缺技能”走向“普遍工具”

GitHub Copilot 早期受控实验显示，在一个 JavaScript HTTP server 任务里，使用 Copilot 的开发者完成更快。这个实验任务范围有限，但它证明了一个方向：AI 可以显著降低某些编码任务的时间成本。

到了 coding agent 阶段，工具已经不只是补全代码。Codex、Claude Code、SWE-agent、OpenHands、Cursor agent、Devin 这类系统都在尝试让模型：

- 读仓库。
- 找相关文件。
- 修改多文件代码。
- 运行测试。
- 根据失败继续迭代。
- 生成 PR 或变更说明。

OpenAI 的 Codex 文档把可复用上下文、测试、review、MCP、skills、automation 都放进最佳实践。Anthropic 的 Claude Code 和 Agent Skills 文档也强调项目知识、流程、工具和权限。这个事实很重要：一线工具厂商自己也没有把 coding agent 理解成“会写代码的模型”，而是理解成“模型 + 上下文 + 工具 + 权限 + 验证 + 状态”的系统。

**事实结论：** “会不会写代码”这件事的门槛正在下降，但“能不能把 agent 接入真实工程系统”反而更重要。

## 事实二：Benchmark 已经从短代码题转向真实工程任务

传统代码 benchmark 常测短函数生成，例如 HumanEval、MBPP。新一代 benchmark 更关心真实工程环境：

| Benchmark | 主要测什么 | 说明了什么 |
|---|---|---|
| SWE-bench | 真实 GitHub issue 修复 | Agent 要理解仓库、定位 bug、改代码、通过测试 |
| SWE-agent | agent-computer interface | 同一个模型在不同工具接口下表现不同 |
| Terminal-Bench | 真实终端任务 | Shell、文件系统、环境配置、命令执行是能力的一部分 |
| LiveCodeBench | 动态代码任务、自修复、执行、测试预测 | 污染控制和任务覆盖面越来越重要 |
| METR long tasks | agent 可可靠完成的任务时长 | 长任务比单步代码生成更接近真实自动化能力 |

这些 benchmark 的共同点是：

```text
它们不再只问“模型能不能写出一段代码”，
而是问“一个完整 agent 系统能不能在环境中完成任务”。
```

这意味着未来的核心能力不只是语言模型能力，还包括：

- 上下文选择。
- 工具接口。
- 编辑协议。
- 测试反馈。
- 状态管理。
- 任务停止条件。
- 失败归因。

**事实结论：** 评测对象正在从 model 转向 model-harness configuration。

## 事实三：AI 对生产力的影响高度依赖任务和组织环境

现有研究并不支持一个简单结论：AI 一定让所有开发者、所有任务都更快。

GitHub Copilot 研究在受控任务里显示明显提速；但 METR 在 2025 年 7 月发布的 RCT 发现，早 2025 的 AI 工具让 16 位熟悉自己大型开源仓库的资深开发者完成任务平均变慢。METR 自己也在 2026 年 2 月更新中提醒，工具能力在快速变化，早 2025 结论不能直接当作长期定律。

DORA 2025 的结论更像工程组织视角：AI 是放大器。高质量工程系统会被放大，混乱流程也会被放大。Stack Overflow 2025 调查则显示，AI 工具使用上升，但开发者对准确性的信任不足；很多人愿意用，但不愿意完全交付信任。

这些来源放在一起看，能得出一个稳定事实：

```text
AI coding 的收益不是自动发生的。
收益取决于任务类型、代码库质量、测试反馈、review 能力、工具熟练度和组织系统。
```

**事实结论：** 壁垒从“个人能不能写代码”转向“个人或组织能不能把 AI 产出变成可靠交付”。

## 事实四：招聘市场已经在为 harness、eval 和 agent reliability 付钱

OpenAI Codex Core Agent 相关岗位明确写到：

- 设计和迭代真实 coding task 上的 agent behavior。
- 做 eval，衡量 performance、regression、failure modes 和 edge cases。
- 通过 prompting、tool-use strategy、context construction 改进表现。
- 分析 production failure，提升 robustness 和 reliability。

这些要求不是“熟悉某门语言语法”，而是：

```text
理解模型行为 + 设计工具和上下文 + 做可复现实验 + 把失败转成系统改进。
```

这很像软件工程能力的重组。过去公司为“会写 React / Go / Java / Rust”付钱；现在顶级 agent 团队还会为“会设计 eval、harness、sandbox、context、tool loop、failure analysis”的人付钱。

**事实结论：** 市场已经开始把 agent reliability、eval 和 harness 当作高价值工程能力。

## 事实五：安全和治理不是附属问题，而是 coding agent 的核心约束

Coding agent 不只是生成文本，它会读文件、写文件、运行命令、访问网络、调用 MCP 工具，甚至操作浏览器和云资源。因此风险面也从“代码有没有 bug”扩展为：

- Prompt injection。
- Secret 泄露。
- 工具过度授权。
- MCP tool poisoning。
- 恶意配置或插件供应链。
- Agent 执行危险命令。
- 误删、误改、误部署。
- 日志和 trace 中暴露敏感信息。

OWASP LLM Top 10 把 prompt injection、supply chain、excessive agency 等列为关键风险。Invariant Labs 的 MCP tool poisoning 研究说明，工具描述和元数据也可能成为攻击入口。CVE-2025-54136 这类漏洞则说明，agent 工具配置和信任机制本身也需要像传统供应链一样治理。

**事实结论：** 当人人都能让 AI 写代码，能不能建立权限、审计、隔离和安全验证会成为新的壁垒。

## 观点层：不同群体其实在说同一件事

把论文、官方文档、工程博客、社区经验和安全报告放在一起，会看到几种声音。

### 观点一：Vibe coding 会扩大入口，但不会自动替代软件工程

自然语言生成代码让更多人能做原型、内部工具和一次性脚本。这是很大的变化。

但低代码、无代码、end-user programming 的历史经验提醒我们：门槛降低之后，常见问题会变成治理、可维护性、集成、权限、安全、可扩展性和所有权。今天的 vibe coding 很像这些历史趋势的更强版本：入口更低，生成更快，但后续质量问题也更容易扩大。

### 观点二：高手把 agent 当执行系统，不当自动负责人

Denny Britz、Addy Osmani、Simon Willison、Jesse Vincent、Anthropic 和 OpenAI 的实践都指向类似模式：

- 人类定义目标和非目标。
- Agent 负责探索、实现、补测试和重复劳动。
- 测试、类型检查、lint、CI、review 是外部反馈。
- 大任务需要计划和状态文件。
- 并行 agent 会提高吞吐，但 review 和合并会成为瓶颈。

高手的差异不在于他们会一条神奇 prompt，而在于他们建立了 agent 可以稳定工作的工程秩序。

### 观点三：企业不会只买“更快写代码”，会买“更低风险交付”

DORA 的视角很关键。AI 让单个开发者输出更多代码，但组织真正关心的是：

- 需求是否更快变成用户价值。
- 变更是否更稳定。
- 事故是否更少。
- 代码是否更容易维护。
- 合规和安全是否可证明。
- 团队是否没有被 review 和返工压垮。

所以企业壁垒不是“用了哪个模型”，而是有没有把 AI 放进测试、CI/CD、平台工程、权限、观测和度量体系里。

### 观点四：安全团队会把 agent 当新的执行主体

传统安全模型里，人写代码、工具执行命令。Agent 时代，模型会根据上下文主动选择工具。于是安全边界必须覆盖：

- Agent 能看到什么。
- Agent 能调用什么。
- 工具返回什么会进入模型上下文。
- 谁批准高风险动作。
- 发生异常时如何追踪 intent、tool call、output 和 policy decision。

这就是 OpenAI “agent-native telemetry” 的价值：不只记录进程做了什么，还记录 agent 为什么做、根据什么上下文做、哪些审批通过或拒绝。

## 推断：壁垒正在迁移到十个层级

下面是本文的核心整理框架。

### 1. 问题定义壁垒

AI 可以帮你写“某个功能”，但不一定知道这个功能是否值得做。

真正稀缺的是：

- 从混乱需求中识别真实问题。
- 判断什么不该做。
- 把模糊愿望拆成可验证目标。
- 理解业务约束、用户场景和失败成本。

如果问题定义错了，代码写得越快，浪费越快。

### 2. 领域知识壁垒

很多高价值软件不是算法题，而是行业知识的编码：

- 金融里的风控、账务、合规。
- 医疗里的流程、术语、隐私。
- 工业里的设备、异常、工单。
- 企业内部的权限、审批、历史包袱。

这些知识不一定公开、不一定结构化、不一定在模型训练语料里。谁拥有高质量领域上下文，谁就拥有更强 agent。

### 3. 规格化壁垒

“请帮我做一个系统”太弱。“目标、非目标、输入输出、边界条件、验收测试、性能要求、安全要求”才是可执行任务。

规格化能力包括：

- 写清楚 spec。
- 定义 done。
- 设计 acceptance criteria。
- 把需求变成测试或 eval。
- 把一次性经验沉淀成模板。

这会变成 AI 时代的核心工程语言。

### 4. 上下文组织壁垒

Agent 不是不知道世界，而是经常不知道你的世界。

上下文壁垒包括：

- 项目文档是否清楚。
- 架构决策是否可追溯。
- 代码约定是否写在 agent 能读到的位置。
- 当前任务状态是否持久化。
- 失败记录是否能被下次复用。
- `AGENTS.md`、skills、plans、status 是否形成层次。

上下文不是聊天附件，而是工程资产。

### 5. Harness 壁垒

Harness 决定 agent 能力是否能落地。

关键组件包括：

- Agent loop。
- Tool surface。
- 文件编辑和 patch 协议。
- Sandbox。
- 权限和 approval。
- MCP / 外部系统连接。
- 任务状态和 session persistence。
- Telemetry。
- Hook / rules / skills。
- Verification loop。

同一个模型，在不同 harness 里会像不同 agent。这就是未来工具竞争和团队能力差异的重要来源。

### 6. 验证壁垒

AI 让生成变便宜，验证就变贵。

验证壁垒包括：

- 会不会写有效测试。
- 会不会设计 eval。
- 会不会判断测试是否覆盖真实风险。
- 会不会做 code review。
- 会不会区分“跑通 demo”和“可上线”。
- 会不会从失败中增加回归测试或 guardrail。

如果没有验证系统，AI 产出的最大问题不是“不工作”，而是“看起来工作”。

### 7. 架构和维护壁垒

Agent 很容易复制局部模式，但不一定维护全局一致性。

架构壁垒包括：

- 模块边界。
- 数据模型。
- API contract。
- 迁移策略。
- 依赖治理。
- 性能和并发。
- 可观测性。
- 技术债管理。

代码越容易生成，长期维护判断越稀缺。

### 8. 安全和治理壁垒

Agent 越能行动，越需要边界。

安全和治理壁垒包括：

- Secret 管理。
- 最小权限。
- 工具白名单和审批。
- 插件 / skill / MCP 供应链审查。
- 运行日志和审计。
- Prompt injection 防护。
- 高风险动作隔离。
- 合规证据。

未来“会用 AI 写代码”和“能安全规模化使用 AI 写代码”会是两件事。

### 9. 交付和运营壁垒

软件不是 merge 之后就结束。

交付和运营壁垒包括：

- CI/CD。
- 灰度发布。
- 回滚。
- 监控。
- incident response。
- 成本控制。
- 支持和文档。
- 用户反馈回流。

Agent 可以让变更更多，但组织必须能承接这些变更。

### 10. 分发和信任壁垒

如果人人能做工具，稀缺的是：

- 谁知道你的工具。
- 谁相信你的工具。
- 谁愿意把数据和流程交给你。
- 谁愿意持续使用。
- 谁愿意为可靠性付费。

这会让品牌、社区、渠道、客户关系、合规资质和生态集成变得更重要。

## 一个可复用判断框架

以后判断某个团队、产品或个人在“人人皆可 coding”时代有没有壁垒，可以用这六个问题。

### 1. 目标是否稀缺

```text
你是否比别人更知道什么值得做、为什么做、做到什么程度算对？
```

低壁垒信号：

- 只会复刻已有产品。
- 需求来自“看到别人做了”。
- 没有明确用户和使用场景。

高壁垒信号：

- 有一手用户反馈。
- 有行业知识。
- 有清楚取舍。
- 知道哪些功能不该做。

### 2. 上下文是否私有且可用

```text
你是否拥有模型默认不知道、但 agent 可以安全使用的上下文？
```

低壁垒信号：

- 每次都靠临时 prompt。
- 文档过期。
- 决策只在人的脑子里。

高壁垒信号：

- 文档、代码、测试、日志、计划互相连接。
- 关键知识版本化。
- Agent 能递进式读取上下文。

### 3. 验证是否可执行

```text
你是否能把“正确”变成测试、eval、review checklist 或可观测指标？
```

低壁垒信号：

- 只看 agent 的文字总结。
- 没有回归测试。
- Review 只看风格，不看行为。

高壁垒信号：

- 需求有验收测试。
- 重要 bug 有回归测试。
- CI 能暴露关键风险。
- 失败会进入新规则或新检查。

### 4. Harness 是否可复用

```text
你是否把重复工作流、规则和工具接入沉淀成 agent runtime 的一部分？
```

低壁垒信号：

- 每个任务从零解释。
- 高风险动作全靠模型自觉。
- 工具接入没有权限边界。

高壁垒信号：

- `AGENTS.md` 简洁但有效。
- 有 task-specific docs / skills / hooks。
- 有 sandbox、approval、telemetry。
- Agent 能在失败后自我修复一轮。

### 5. 交付系统是否能承接高吞吐

```text
如果 AI 让代码变更多 3 倍，你的测试、review、发布和监控会变好还是崩掉？
```

低壁垒信号：

- 测试慢且不可信。
- Review 堆积。
- 发布依赖手工英雄主义。
- 事故复盘不产生系统改进。

高壁垒信号：

- 小批量变更。
- 自动化测试和部署。
- 快速回滚。
- 事故复盘能更新 harness。

### 6. 信任是否可积累

```text
用户、团队或组织是否越来越相信你的系统，而不只是被 demo 吸引？
```

低壁垒信号：

- 只有漂亮 demo。
- 出错后无法解释。
- 数据、权限、责任边界不清。

高壁垒信号：

- 行为可追踪。
- 质量可度量。
- 风险可隔离。
- 用户反馈能进入下一轮改进。

## 不同角色的壁垒会不一样

### 对个人开发者

最大壁垒不是记住更多 API，而是：

- 会拆任务。
- 会写测试。
- 会 review AI 代码。
- 会维护自己的知识库和 workflow。
- 会判断什么该让 agent 做，什么该自己想。

个人开发者的复利来自“把每次踩坑变成下次 agent 能读到的上下文、测试或技能”。

### 对工程团队

最大壁垒是把 AI 放进已有工程系统：

- 项目结构清楚。
- 本地环境好启动。
- 测试和 CI 可靠。
- Code review 标准一致。
- 权限和 secret 边界明确。
- 文档和计划不脱离代码。

团队不是只要买工具，而是要让工具进入可控流程。

### 对创业者

最大壁垒不是“我能不能做出 app”，而是：

- 有没有真实用户问题。
- 有没有分发渠道。
- 有没有数据和 workflow 入口。
- 有没有信任和品牌。
- 有没有让产品越用越好的反馈闭环。

AI 会让 MVP 变便宜，也会让同质化更严重。分发、信任和领域嵌入会更关键。

### 对平台和工具厂商

最大壁垒是 harness 和生态：

- Agent loop 是否可靠。
- Context engine 是否强。
- Tool interface 是否好用。
- Sandbox 和 approval 是否安全。
- Trace 和 eval 是否可复现。
- 插件、skills、MCP 生态是否可治理。
- 能否把真实使用反馈回流到模型和产品。

模型会升级，但 harness 会积累用户工作流和工程惯性。

## 对本仓库的启发

这个问题最终会落回本仓库的主题：

```text
人人皆可 coding 之后，人人都需要 harness。
```

对当前文档体系，可以继续沉淀这些方向：

- `AGENTS.md`：保持入口地图，不写成百科全书。
- `docs/coding-agents/agent/playbooks/`：继续沉淀可复制工作流。
- `docs/coding-agents/research/`：继续研究 agent hooks、skills、subagents、review、plan docs。
- `docs/ai-models/`：不要只追模型排名，要解释 benchmark 测什么。
- `docs/harness-engineering/`：把上下文、工具、权限、验证、状态、反馈闭环系统化。
- 未来 `examples/`：可以做最小 agent harness demo，例如任务计划、测试反馈、hook、日志审计和权限边界。

最值得继续研究的不是“哪个模型最强”，而是：

```text
什么样的 harness 能把不同模型稳定变成可靠工程产出？
```

## 可复用调研 Prompt

以后要让 agent 继续调研类似问题，可以直接用这个 prompt：

```text
请围绕“人人皆可 coding 的时代，真正的壁垒是什么”做一份调研。

资料范围包括：
- 论文：代码生成、软件工程 agent、end-user programming、low-code/no-code、agent safety。
- 技术报告：模型公司、METR、DORA、GitHub、Stack Overflow、企业采用报告。
- 官方文档：Codex、Claude Code、GitHub Copilot、Cursor、Devin、OpenHands 等工具的 agent/harness/安全文档。
- Benchmark / eval：SWE-bench、Terminal-Bench、LiveCodeBench、Aider Polyglot、METR long tasks 等。
- 开源项目：coding agent、agent runtime、eval harness、MCP server、skills/hooks/plugin 生态。
- 开发者社区讨论：Hacker News、Reddit、GitHub Issues/Discussions、作者原文和一线工程师博客。
- 工程案例：公司如何真实使用 coding agent，哪些场景提效，哪些场景失败。
- 招聘市场：AI coding、agent harness、eval、developer tools、platform engineering 相关岗位要求。
- 事故复盘和安全报告：prompt injection、MCP/tool poisoning、secret 泄露、权限误用、AI 生成代码漏洞。
- 历史类比：低代码/无代码、Excel/end-user programming、DevOps、云计算、开源和自动化测试。

要求：
1. 区分事实、观点和推断。
2. 对可能变化的信息标明查询日期。
3. 优先引用官方来源、论文、项目主页和作者原文。
4. 不要把单个 benchmark 排名写成长期结论，要解释它测什么、局限是什么。
5. 最后总结成可复用判断框架，并说明对个人开发者、工程团队、创业者和工具平台分别意味着什么。
```

## 一句话总结

“人人皆可 coding”的时代，低层语法壁垒下降，但高层工程壁垒上升。

真正的护城河不是写出代码，而是：

```text
定义正确问题，
组织正确上下文，
建立正确反馈，
交付正确系统，
并让这个过程不断复利。
```

这就是 coding agent 时代的软件工程核心能力。
