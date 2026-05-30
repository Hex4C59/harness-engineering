# Tool Use 调研：从函数调用到 Agent Action Layer

调研日期：2026-05-30

## 这篇文档回答什么

这里的 **Tool Use** 不只指 OpenAI / Anthropic / Gemini API 里的 `function calling`，而是指模型或 agent 通过外部能力获取信息、执行计算、修改状态、控制软件环境、调用业务系统，并把执行结果纳入下一步推理的完整机制。

调研范围覆盖：

- 论文、技术报告和 survey。
- 官方 API / SDK / agent 文档。
- benchmark / eval。
- 开源项目。
- 开发者社区讨论。
- 工程案例。
- 招聘市场信号。
- 安全事故、攻击复盘和历史类比。

这篇文档会显式区分：

- **事实**：来源中可直接确认的内容。
- **观点**：来源作者、社区或本文整理出的判断。
- **推断**：基于多个来源交叉后的工程判断，不等同于已被证明的事实。

核心结论：

> Tool Use 正在从“模型能不能输出 JSON 函数调用”演化为一整层 agent action infrastructure：工具定义、工具发现、工具选择、参数生成、执行隔离、权限审批、状态观测、失败恢复、安全审计和 eval，都属于同一个工程问题。

## 快速结论

| 类型 | 结论 |
|---|---|
| 事实 | 主流模型平台已经把工具能力产品化：OpenAI 提供 built-in tools、function calling、MCP、shell / computer use 等工具入口；Anthropic 提供 Claude tool use、computer use 和 MCP；Google Gemini API 提供 function calling、code execution、Google Search 等工具。 |
| 事实 | 工具调用 benchmark 已经从单轮 API 选择扩展到多轮交互、状态依赖、浏览器、桌面、终端、软件工程和 ML 工程环境，例如 BFCL、ToolSandbox、τ-bench、WebArena、OSWorld、Terminal-Bench、SWE-bench、MLE-bench。 |
| 事实 | 安全资料反复指出：一旦 LLM 可以调用工具，prompt injection 的影响会扩大到这些工具 / API 能做的最坏动作；OWASP 和 NCSC 都把 prompt injection、excessive agency、tool misuse 作为核心风险。 |
| 观点 | Anthropic 的工程博客把“高质量工具 + eval”视为 agent 性能的关键，而不是只优化 prompt；OpenAI Codex harness 的材料也把工具执行、approval、sandbox、trace、typed events 看成 agent runtime 的核心。 |
| 社区观点 | 开发者社区反复抱怨的问题不是“没有工具”，而是工具太多、工具描述不清、MCP server 暴露面太大、工具权限难分层、模型有时不调用该调用的工具或乱调用。 |
| 推断 | 未来真正稀缺的不是“会写 prompt 的人”，而是能设计工具边界、权限模型、执行环境、观测系统和 eval loop 的工程师。 |
| 推断 | Tool Use 的长期评估对象不应是单个模型，而应是 `model + tool schema + harness + environment + policy + evaluator` 的组合。 |

## 概念边界

### Tool Use 包含什么

**事实**

官方文档和工程实践里，Tool Use 至少包含这些形态：

- **Function calling / custom tools**：模型选择一个开发者定义的函数，并生成结构化参数。
- **Built-in tools**：平台托管的 web search、file search、code execution、image generation、computer use 等。
- **Shell / code execution**：模型通过受控环境运行命令或代码。
- **Computer use / browser use**：模型通过视觉、DOM、accessibility tree 或 GUI action 控制软件界面。
- **MCP tools**：通过 Model Context Protocol 暴露外部工具、数据源和业务系统。
- **Agent-as-tool**：一个 specialist agent 被包装成另一个 agent 可调用的工具。
- **Skills / rule bundles**：不是传统 API，但会影响 agent 怎样调用工具、执行流程和处理失败。

OpenAI 的工具文档把 built-in tools、function calling、tool search、remote MCP server 放在同一类“扩展模型能力”的入口下；Anthropic 的 MCP 发布文章把 MCP 定位为连接 AI assistants 和外部系统的开放标准；Google Gemini API 的 function calling 文档则明确把工具用于外部知识、能力扩展和真实动作。

参考：

- OpenAI: [Using tools](https://developers.openai.com/api/docs/guides/tools)
- Anthropic: [How to implement tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools)
- Anthropic: [Introducing the Model Context Protocol](https://www.anthropic.com/news/model-context-protocol)
- Google: [Function calling with the Gemini API](https://ai.google.dev/gemini-api/docs/function-calling)
- Google: [Using Tools & Agents with Gemini API](https://ai.google.dev/gemini-api/docs/tools)

### 一个完整 Tool Use 生命周期

**推断**

工程上不应只看“模型有没有吐出函数名”。一个完整工具调用链路更接近：

```text
用户意图
  -> 上下文选择
  -> 工具发现 / 工具裁剪
  -> 工具选择
  -> 参数生成
  -> policy / approval / sandbox 检查
  -> 工具执行
  -> observation 写回上下文
  -> 状态更新
  -> 验证 / retry / fallback
  -> trace / audit / eval
```

这意味着 Tool Use 是 harness engineering 的核心模块之一。函数签名只是接口，真正决定可靠性的通常是工具是否可理解、权限是否合理、输出是否高信号、错误是否可恢复、执行是否可审计。

## 研究脉络

### 1. 从 ReAct 到 Toolformer：让模型边想边行动

**事实**

ReAct 提出让 LLM 交替生成 reasoning trace 和 action，使模型可以一边推理一边访问外部环境或知识源。Toolformer 则展示了模型可以通过自监督方式学习什么时候调用 API、调用什么参数、如何把结果纳入后续 token 预测。两者奠定了后续 tool-using agent 的两个关键方向：

- ReAct：推理和行动交错。
- Toolformer：工具调用能力可以被训练。

参考：

- [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)
- [Toolformer: Language Models Can Teach Themselves to Use Tools](https://arxiv.org/abs/2302.04761)

**推断**

ReAct 更像 runtime pattern，Toolformer 更像 training pattern。今天的 agent 系统通常把两者混合：模型在 runtime 里用 ReAct-like loop 行动，同时模型本身也在 post-training 中学习工具调用格式和策略。

### 2. 从小工具到真实 API 生态

**事实**

2023 年以后，研究开始从 calculator / search 这类小工具转向真实 API 生态：

- API-Bank 提供 tool-augmented LLM benchmark。
- Gorilla / APIBench 聚焦把自然语言请求映射到大量 API 调用。
- ToolLLM / ToolBench 收集 16,464 个 RapidAPI RESTful APIs，覆盖 49 个类别，并构建 instruction / solution path 数据。
- Tool Learning survey 把工具学习流程拆成 task planning、tool selection、tool calling、response generation 等阶段。

参考：

- [API-Bank: A Comprehensive Benchmark for Tool-Augmented LLMs](https://arxiv.org/abs/2304.08244)
- [Gorilla: Large Language Model Connected with Massive APIs](https://arxiv.org/abs/2305.15334)
- [ToolLLM: Facilitating Large Language Models to Master 16000+ Real-world APIs](https://arxiv.org/abs/2307.16789)
- [Tool Learning with Large Language Models: A Survey](https://arxiv.org/abs/2405.17935)
- [Tool Learning with Foundation Models](https://arxiv.org/abs/2304.08354)

**推断**

真实 API 生态暴露出一个简单事实：工具使用不是“会写 JSON”就够了。agent 还要理解 API 之间的依赖、状态、错误码、参数语义、权限、业务规则和返回结果。

### 3. 从无状态单轮到有状态交互

**事实**

ToolSandbox 明确指出，很多早期工具 benchmark 只测试无状态 web service、单轮 prompt 或离线轨迹，而真实工具使用常常有状态依赖、用户交互和动态评估。τ-bench 则把工具、agent 和模拟用户放到 airline / retail 这类真实领域交互中评估可靠性。

参考：

- [ToolSandbox: A Stateful, Conversational, Interactive Evaluation Benchmark for LLM Tool Use Capabilities](https://arxiv.org/abs/2408.04682)
- [τ-bench](https://www.tau-bench.com/)
- [τ-bench: A Benchmark for Tool-Agent-User Interaction in Real-World Domains](https://arxiv.org/abs/2406.12045)

**推断**

Tool Use 的难点正在从“函数调用准确率”转向“长期任务完成率”。多轮、状态、用户澄清、工具副作用和错误恢复，比单次 function name / argument match 更接近生产问题。

## 官方平台实践

### OpenAI

**事实**

OpenAI 在 2023-06 发布 function calling；2024-08 发布 Structured Outputs，可在 function definition 中设置 `strict: true` 来提高 schema adherence；2025-03 发布 Responses API、Agents SDK、web search、file search、computer use 等 agent building tools。OpenAI 当前工具文档把工具分为 function calling、web search、MCP、skills、shell、computer use、image generation、file search、tool search 等。

参考：

- [Function calling and other API updates](https://openai.com/index/function-calling-and-other-api-updates/)
- [Introducing Structured Outputs in the API](https://openai.com/index/introducing-structured-outputs-in-the-api/)
- [New tools for building agents](https://openai.com/index/new-tools-for-building-agents/)
- [Computer-Using Agent](https://openai.com/index/computer-using-agent/)
- [Using tools](https://developers.openai.com/api/docs/guides/tools)

**事实**

Codex App Server / harness 文章把 Codex 的 agent run 拆成 typed items，例如 user message、agent message、tool execution、approval request、diff，并把 agent loop、tool execution、approval、sandbox、thread lifecycle 做成可复用协议层。

参考：

- [Unlocking the Codex harness: how we built the App Server](https://openai.com/index/unlocking-the-codex-harness/)
- [Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/)

**推断**

OpenAI 的方向说明：Tool Use 正在从 API 参数变成产品基础设施。工具调用需要客户端协议、服务端 run lifecycle、approval、sandbox 和 trace 一起设计。

### Anthropic

**事实**

Anthropic 的 Claude API 支持 client tools 和 Anthropic-defined tools。其 computer use 文档明确说明：应用必须显式执行工具，Claude 不能直接运行工具。Anthropic 在 2024-11 发布 MCP，用于标准化 AI assistant 与外部数据源和工具的连接。

参考：

- [How to implement tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools)
- [Computer use tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool)
- [Introducing the Model Context Protocol](https://www.anthropic.com/news/model-context-protocol)

**观点**

Anthropic 在《Writing effective tools for AI agents》中的核心判断是：agent 的效果高度依赖工具质量。文章建议通过真实任务 eval 迭代工具，选择清晰、不重叠的工具，给工具命名空间，返回高信号上下文，优化 token 效率，并把工具描述写得像给新员工 onboarding。

参考：

- [Writing effective tools for AI agents - using AI agents](https://www.anthropic.com/engineering/writing-tools-for-agents)
- [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

**推断**

Anthropic 的实践把“工具设计”从后端 API 设计扩展成一种 agent interface design：工具不是给人类 SDK 用户看的，而是给概率模型调用的。描述、参数名、返回值、错误文本都会进入模型决策面。

### Google

**事实**

Gemini API 的 function calling 文档把工具调用归纳为三类用途：访问外部知识、扩展模型能力、执行真实动作。Gemini API 支持 `AUTO`、`ANY`、`NONE` 等 function calling mode；code execution tool 允许模型生成并运行 Python 代码，并可与 Google Search grounding 组合。

参考：

- [Function calling with the Gemini API](https://ai.google.dev/gemini-api/docs/function-calling)
- [Code execution](https://ai.google.dev/gemini-api/docs/code-execution)
- [Agent Development Kit tools](https://adk.dev/tools/)

**推断**

Google 的设计体现了另一个趋势：工具不只用于调用业务 API，也用于把模型变成“可执行思考”的系统，例如通过 code execution 做计算、数据处理和迭代验证。

### LangChain / LlamaIndex 等框架

**事实**

LangChain 把工具定义为有输入输出契约的 callable function，并由模型根据上下文决定是否调用；LangSmith traces 记录 agent 执行的每一步，包括 tool calls、model interactions 和 decision points。LlamaIndex 也提供 FunctionAgent / ReActAgent 和 Tool Specs 来接入外部服务。

参考：

- LangChain: [Tools](https://docs.langchain.com/oss/python/langchain-tools)
- LangChain: [LangSmith Observability](https://docs.langchain.com/oss/python/langchain/observability)
- LlamaIndex: [Agents](https://docs.llamaindex.ai/en/stable/module_guides/deploying/agents/)

**推断**

框架正在把工具调用抽象成标准 agent node / tool node / trace span。未来工具治理很可能发生在这些中间层，而不是每个业务 API 内部。

## Benchmark / Eval 地图

| Benchmark / Eval | 主要测什么 | 对 Tool Use 的启发 | 局限 |
|---|---|---|---|
| [BFCL](https://gorilla.cs.berkeley.edu/leaderboard) | function calling / tool calling 准确率 | 适合比较模型对 schema、参数和多函数调用的基础能力 | 不等同于完整业务任务成功率 |
| [API-Bank](https://arxiv.org/abs/2304.08244) | tool-augmented LLM 的 API 使用 | 早期系统化工具 benchmark | 更偏 API 层，不覆盖复杂环境控制 |
| [ToolLLM / ToolBench](https://arxiv.org/abs/2307.16789) | 大规模真实 API 选择和调用 | 说明工具检索、API 选择和调用路径是独立问题 | RapidAPI 任务和真实生产权限 / SLA 仍有距离 |
| [ToolSandbox](https://arxiv.org/abs/2408.04682) | 有状态、多轮、交互式工具调用 | 把状态依赖、canonicalization、信息不足纳入评估 | 仍是受控模拟环境 |
| [τ-bench](https://www.tau-bench.com/) | 工具、agent、用户交互的任务可靠性 | 更接近客服 / ops 场景中的多轮任务 | 覆盖领域有限 |
| [WebArena](https://webarena.dev/) | 真实网站环境中的 web agent | 测浏览器行动、长程任务和环境反馈 | 初始结果显示强模型和人类仍有明显差距 |
| [OSWorld](https://arxiv.org/abs/2404.07972) | 真实桌面 / OS 环境中的 computer use | 测 GUI grounding、跨应用工作流和文件 I/O | 成本高，复现和稳定性比纯 API benchmark 难 |
| [WorkArena](https://arxiv.org/abs/2403.07718) | 企业软件中的知识工作任务 | 把 tool use 放进 ServiceNow 这类复杂业务 UI | 依赖特定企业平台 |
| [SWE-bench](https://www.swebench.com/) / [SWE-agent](https://arxiv.org/abs/2405.15793) | GitHub issue 到 patch 的软件工程任务 | 工具包括读写文件、搜索代码、运行测试、提交 patch | 分数受 scaffold / harness 强烈影响 |
| [Terminal-Bench](https://www.tbench.ai/) | 真实终端任务的 end-to-end 完成率 | 把 shell、文件、服务、测试验证放进同一执行环境 | benchmark harness 本身也可能被 reward hacking |
| [MLE-bench](https://openai.com/index/mle-bench/) | Kaggle 风格 ML 工程任务 | 测实验、数据处理、训练、提交和资源管理 | 成本高，运行条件对结果影响大 |

**推断**

工具 eval 正在从“选对函数”变成“在受控环境中完成工作”。因此报告结果时，应尽量说明：

```text
model
+ scaffold / harness
+ visible tools
+ tool descriptions
+ permission policy
+ retries / parallelism
+ environment
+ evaluator
```

单独说“某模型 Tool Use 很强”越来越不够严谨。

## 开源项目和工程案例

### SWE-agent：Agent-Computer Interface

**事实**

SWE-agent 论文提出 custom agent-computer interface 可以显著增强 agent 创建 / 编辑代码、浏览仓库、运行测试和程序的能力。

参考：

- [SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering](https://arxiv.org/abs/2405.15793)

**推断**

对 coding agent 来说，`read_file`、`edit_file`、`search`、`run_tests` 这类工具不是中性接口。文件编辑粒度、搜索输出格式、错误展示方式都会改变模型能力。

### OpenHands

**事实**

OpenHands / OpenDevin 是开源软件开发 agent 平台，目标是让 agent 像人类开发者一样写代码、使用命令行、浏览网页。相关论文和仓库都强调 terminal、browser、file editing、workspace 等工具面。

参考：

- [OpenHands paper](https://arxiv.org/abs/2407.16741)
- [OpenHands GitHub](https://github.com/OpenHands/OpenHands)

**推断**

OpenHands 代表了开源 agent runtime 的典型形态：模型只是一个组件，真正的产品差异来自 workspace、工具、sandbox、UI、trace、human control 和可复现运行。

### browser-use

**事实**

browser-use 是开源浏览器自动化框架，目标是让网站可被 AI agent 操作，提供浏览器、动作和自定义工具接口。

参考：

- [browser-use GitHub](https://github.com/browser-use/browser-use)

**推断**

Browser tool 的工程难点与 API tool 不同：页面状态、DOM 变化、登录态、验证码、动态 UI、误点击和不可逆动作都让“工具调用成功”更难定义。

### MCP 生态

**事实**

MCP 官方 GitHub 维护协议规范和文档。Anthropic 后续宣布将 MCP 捐给 Linux Foundation 下的 Agentic AI Foundation，并推动官方 registry、server identity、asynchronous operations 等方向。

参考：

- [MCP GitHub](https://github.com/modelcontextprotocol/modelcontextprotocol)
- [MCP Security Best Practices](https://modelcontextprotocol.io/specification/2025-06-18/basic/security_best_practices)
- [MCP tool annotations](https://blog.modelcontextprotocol.io/posts/2026-03-16-tool-annotations/)

**推断**

MCP 解决的是“工具连接协议”的标准化，但不自动解决工具质量、安全边界、权限最小化、工具发现和上下文污染。协议只是 action layer 的一部分。

### Stripe Minions

**事实**

Stripe 的 Minions 工程博客描述了 unattended coding agents、cloud devbox、blueprint、Toolshed 和 scoped tools。Part 2 明确提到 agents 在“更小的盒子”和精心裁剪的工具集合里表现更好。

参考：

- [Minions: Stripe's one-shot, end-to-end coding agents - Part 2](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2)

**观点**

Stripe 的经验接近一个生产级判断：不要把全部工具都暴露给 agent。按任务裁剪工具、把 deterministic code 和 agent loop 混合，比“给模型一个巨大的工具箱”更可靠。

## 开发者社区讨论

**事实 / 弱信号**

社区讨论中反复出现以下问题：

- MCP server 暴露工具过多后，模型选择工具变差或上下文被工具 schema 挤占。
- 很多工具描述只说明“这个 API 做什么”，没有说明“什么时候该用它、什么时候不该用它”。
- 开发者希望按调用上下文限制工具可见性，而不是让 agent 永远看到全部工具。
- 工具调用、结构化输出、并行调用在不同 provider / framework 中存在兼容性和行为差异。
- Human approval 如果只展示 agent 的自然语言摘要，而不展示真实 tool call、参数和影响范围，容易形成虚假的安全感。

参考社区线索：

- [98% of MCP Tools Don't Tell AI Agents When to Use Them](https://dev.to/spiderrating/98-of-mcp-tools-dont-tell-ai-agents-when-to-use-them-28a8)
- [MCP server integration degrades performance with too many integrations](https://www.linkedin.com/posts/arindam2004_every-mcp-server-you-add-makes-your-agent-activity-7459960324675395587-IUEC)
- [Are AI agent tools like MCP servers too fragmented right now?](https://www.reddit.com/r/LocalLLaMA/comments/1sqif6v/are_ai_agent_tools_like_mcp_servers_too/)
- [MCP servers give agents tool access. We measured what happens when nothing enforces the boundary.](https://www.reddit.com/r/mcp/comments/1rlieo1/mcp_servers_give_agents_tool_access_we_measured/)
- [Model context protocol security questions for enterprise developer tools](https://www.reddit.com/r/ContextEngineering/comments/1sz0vyz/model_context_protocol_security_questions_for/)

**注意**

社区讨论不能当作严格事实，只能当作问题发现来源。它们的价值在于暴露真实开发者卡点：工具太多、边界不清、行为不稳、权限难管。

**推断**

工具系统需要类似“API product management”的工作：命名、描述、版本、权限、deprecation、兼容性、观测、用户教育和安全 review。

## 安全事故和风险复盘

### Prompt injection + tool use = confused deputy

**事实**

NCSC 指出，当 LLM 输出会触发工具 / API 时，prompt injection 的影响会扩大到攻击者直接访问这些工具 / API 的最坏场景。OWASP LLM Top 10 也把 Prompt Injection 和 Excessive Agency 列为核心风险。

参考：

- NCSC: [Prompt injection is not SQL injection](https://www.ncsc.gov.uk/blog-post/prompt-injection-is-not-sql-injection)
- OWASP: [Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications)
- OWASP: [MCP Tool Poisoning](https://owasp.org/www-community/attacks/MCP_Tool_Poisoning)

**推断**

对 agent 来说，prompt injection 不是文本安全问题，而是权限委托问题。模型一旦能执行动作，就可能成为 confused deputy：攻击者通过模型间接使用本不该给攻击者的工具权限。

### ChatGPT plugins / markdown image exfiltration

**事实**

2023 年，研究者 Johann Rehberger / Embrace The Red 展示了通过间接 prompt injection 诱导 LLM 输出 markdown image，从而把对话数据编码进图片 URL 并触发浏览器请求的 exfiltration 方式。Embrace The Red 也记录了 Anthropic Claude 的类似 markdown image exfiltration 漏洞及后续修复：不再自动渲染图片，而是让用户点击显示。

参考：

- MITRE ATLAS case mirror: [ChatGPT Conversation Exfiltration](https://www.startupdefense.io/mitre-atlas-case-studies/aml-cs0021-chatgpt-conversation-exfiltration)
- Embrace The Red: [Anthropic Claude Data Exfiltration Vulnerability Fixed](https://embracethered.com/blog/posts/2023/anthropic-fixes-claude-data-exfiltration-via-images/)
- Tom's Hardware: [ChatGPT Plugins Open Security Holes From PDFs, Websites and More](https://www.tomshardware.com/news/chatgpt-plugins-prompt-injection)

**推断**

这类事件说明，工具的“输出渲染方式”也是工具安全边界的一部分。即使没有显式 HTTP tool，markdown rendering、image loading、browser side effects 也可能成为隐式工具。

### MCP Tool Poisoning / Rug Pull

**事实**

MCP security best practices 提醒客户端关注工具列表变化、危险命令模式、权限确认和 tool call approval。OWASP 描述了 MCP Tool Poisoning：攻击者把恶意指令藏在 tool descriptions 中，让 agent 在使用工具时被间接注入。后续研究还讨论 tool poisoning、shadowing、rug pull 等攻击形态。

参考：

- MCP: [Security Best Practices](https://modelcontextprotocol.io/specification/2025-06-18/basic/security_best_practices)
- OWASP: [MCP Tool Poisoning](https://owasp.org/www-community/attacks/MCP_Tool_Poisoning)
- [Securing the Model Context Protocol](https://arxiv.org/abs/2512.06556)
- [MCPTox: A Benchmark for Tool Poisoning Attack on Real-World MCP Servers](https://arxiv.org/abs/2508.14925)
- [Systematic Analysis of MCP Security](https://huggingface.co/papers/2508.12538)

**推断**

MCP 把 tool metadata 变成了模型上下文的一部分，因此 tool description 本身就是供应链输入。传统 API 文档出错只会让人困惑；agent 工具描述被污染，可能直接改变工具执行行为。

### 最小权限和审批不应只靠提示词

**观点 / 推断**

实践上需要把工具风险分层：

- 只读工具默认可自动执行。
- 会写入业务系统、发送消息、扣款、部署、删除、提交代码的工具需要审批。
- shell、browser、computer use 应放入 sandbox 或隔离 workspace。
- approval UI 应展示真实工具名、参数、目标对象、影响范围，而不是只展示 agent 总结。
- 审批、拦截、审计应在 harness / host 层实现，而不是只让模型“承诺不要乱用”。

## 招聘市场信号

**事实**

2026 年前后的 AI agent 岗位描述已经把 Tool Use 写成明确能力：

- OpenAI Frontier Evals & Environments 岗位描述覆盖 coding、tool use、computer use、multi-agent coordination、long-horizon execution、evals 等。
- Apple Multimodal AI 岗位描述提到 context optimization、multi-turn orchestration、tool use、planning、evaluation、robustness。
- JetBrains Agentic Models 岗位描述提到 multi-step coding agents、planning、tool use、agent workflows、evaluation pipelines。
- ByteDance Agent Systems / AI Coding Environment 岗位描述提到 agent harness、tool integration、sandboxing、planning、tool use、memory、coordination、benchmarking。
- Anthropic Tool Use Safety 岗位描述聚焦 prompt injection、data exfiltration、tool misuse、多轮 agent 对话和大规模工具访问的安全。

参考：

- OpenAI Careers: [Research Engineer, Frontier Evals & Environments](https://openai.com/careers/research-engineer-frontier-evals-and-environments-san-francisco/)
- Apple Jobs: [Applied Research Engineer - Multimodal AI](https://jobs.apple.com/en-ng/details/200649931/applied-research-engineer-multimodal-ai)
- JetBrains: [Research Engineer (Agentic Models)](https://agentic-engineering-jobs.com/jobs/jetbrains-research-engineer-agentic-models-hjA7hu)
- ByteDance: [Agent Systems & AI Coding Environment role](https://www.ziprecruiter.com/c/ByteDance/Job/Research-Engineer-Graduate-%28Agent-Systems-%26-AI-Coding-Environment-Seed-Infra%29-2026-Start-%28PhD%29/-in-Seattle%2CWA?jid=7ed5b1a1bb7497b9)
- Anthropic: [Research Engineer / Scientist, Tool Use Safety](https://jobs.generalcatalyst.com/companies/anthropic/jobs/59282574-research-engineer-scientist-tool-use-safety)

**推断**

招聘市场正在把“Tool Use”拆成三类能力：

- **Agent runtime / harness 工程**：工具执行、sandbox、权限、状态、workflow。
- **Agent eval / environment 工程**：构造任务环境、grader、trace、benchmark。
- **Tool use safety**：prompt injection、data exfiltration、tool misuse、least privilege。

这比传统 prompt engineering 更接近系统工程、平台工程和安全工程。

## 历史类比

### Unix pipes：小工具组合的力量和边界

**事实**

Unix pipes 和“做一件事并做好”的工具哲学说明：强大的系统可以由小工具组合而成。

参考：

- [Pipeline (Unix)](https://en.wikipedia.org/wiki/Pipeline_%28Unix%29)
- [Doug McIlroy / Unix philosophy](https://en.wikiquote.org/wiki/Doug_McIlroy)

**类比**

Agent tool use 像 Unix pipe，但多了一个不确定的 planner。Unix shell 里的组合由人写；agent 里的组合由模型动态选择。因此工具必须更自描述、更可观测、更容易回滚。

### SQL injection：指令和数据混淆

**事实**

OWASP SQL injection 资料把问题归因于用户输入被解释成 SQL 命令的一部分，并建议 prepared statements / parameterized queries。

参考：

- OWASP: [SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)

**类比**

Prompt injection 和 SQL injection 都涉及“数据变成指令”。区别是 SQL 最终通过参数化建立较强边界，而 LLM 当前没有同等强度的语义边界。因此 Tool Use 的防线必须移到工具权限、host policy、sandbox 和审批层。

### Browser extensions：权限过大和供应链

**事实**

浏览器扩展安全研究长期关注 least privilege、privilege separation、extension permission 和恶意扩展。

参考：

- Google Research: [Protecting Browsers from Extension Vulnerabilities](https://research.google/pubs/protecting-browsers-from-extension-vulnerabilities/)
- Microsoft Research: [Verified Security for Browser Extensions](https://www.microsoft.com/en-us/research/publication/verified-security-for-browser-extensions/)

**类比**

MCP server 和 agent tools 很像浏览器扩展：它们扩展宿主能力，也带来权限、供应链、更新和用户审批问题。一个“看起来只是读文件”的工具，如果能和网络发送工具组合，就可能变成 exfiltration pipeline。

## 实践框架：如何设计一个 agent tool

**推断**

设计工具时，建议写一张 tool card：

```text
工具名：
一句话用途：
什么时候使用：
什么时候不要使用：
输入 schema：
输出 schema：
是否读外部世界：
是否读取敏感数据：
是否写入 / 删除 / 发送 / 部署：
是否幂等：
是否可回滚：
权限级别：
是否需要 human approval：
失败模式：
错误输出规范：
最大输出 token：
日志 / trace 字段：
eval 任务：
安全测试：
owner：
版本：
```

### 工具设计 checklist

- 工具名要表达业务动作，而不是内部实现细节。
- 参数名要明确，例如 `customer_id` 优于 `user`。
- 工具描述要说明何时使用、何时不用、前置条件和副作用。
- 优先给 agent 高层工具，例如 `search_logs` 优于 `read_all_logs`。
- 返回高信号摘要和必要 ID，不要把整页 HTML、完整日志或巨大 JSON 原样塞回上下文。
- 对 destructive / open-world 工具加明确 annotation 和 approval。
- 将 read / draft / execute 拆开，例如 `draft_email` 和 `send_email` 不应是同一个自动工具。
- 工具输出要包含可验证状态，例如 created object id、diff、test result、commit hash。
- 工具失败要返回可操作错误，而不是只抛栈。
- 对多工具组合做 eval，不只测试单个工具。
- 对工具描述做 prompt injection / poisoning 扫描。
- 记录每次 tool call 的输入、输出、耗时、成本、审批和调用者上下文。

## 未解决问题

**推断**

Tool Use 还有这些开放问题：

- **工具发现**：当 agent 可用工具从 10 个扩展到 1000 个时，如何动态裁剪而不遗漏关键工具？
- **工具语义**：如何让模型理解业务工具的前置条件、隐含约束和副作用？
- **权限模型**：如何把用户权限、agent 权限、工具权限、数据权限、环境权限统一起来？
- **跨工具安全**：单个工具安全不代表组合安全，如何检测 `read_secret + send_http` 这类组合风险？
- **可恢复执行**：长任务中某个工具失败后，agent 应重试、降级、回滚还是请求人类？
- **评价对象**：leaderboard 应该评模型、工具 schema、agent scaffold，还是完整 runtime？
- **工具版本**：工具描述、参数、权限变化后，如何让历史 trace 可回放、eval 可比较？
- **代码模式 vs 工具模式**：让 agent 写代码调用 API，有时比给它 100 个窄工具更稳；什么时候该给工具，什么时候该给 code execution？

## 对本项目的启发

**推断**

对 harness engineering 主题而言，Tool Use 应作为独立层设计：

```text
Model
  -> Context
  -> Tools
  -> Policy
  -> Execution Environment
  -> Observation
  -> Eval
  -> Feedback
```

如果只优化 prompt，而工具边界混乱、权限过宽、输出噪声大、失败不可观测，agent 可靠性不会稳定提升。

更具体地说：

- `AGENTS.md` / docs 应帮助 agent 找到正确工具和流程，而不是堆满全部细节。
- 工具要有 owner、schema、权限、示例和 eval。
- 每个工具调用都应进入 trace。
- 高风险工具必须有 host-layer approval，而不是 prompt-layer 承诺。
- benchmark 要记录 harness 配置，否则模型分数不可解释。
- 新增 MCP server 前，应先问“这个任务真的需要暴露这些工具吗？”

## 参考资料

### Survey / 论文

- [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)
- [Toolformer: Language Models Can Teach Themselves to Use Tools](https://arxiv.org/abs/2302.04761)
- [Tool Learning with Foundation Models](https://arxiv.org/abs/2304.08354)
- [Tool Learning with Large Language Models: A Survey](https://arxiv.org/abs/2405.17935)
- [API-Bank](https://arxiv.org/abs/2304.08244)
- [Gorilla](https://arxiv.org/abs/2305.15334)
- [ToolLLM / ToolBench](https://arxiv.org/abs/2307.16789)
- [ToolSandbox](https://arxiv.org/abs/2408.04682)
- [MCP-AgentBench](https://arxiv.org/abs/2509.09734)

### 官方文档 / 工程博客

- OpenAI: [Using tools](https://developers.openai.com/api/docs/guides/tools)
- OpenAI: [Function calling and other API updates](https://openai.com/index/function-calling-and-other-api-updates/)
- OpenAI: [Introducing Structured Outputs in the API](https://openai.com/index/introducing-structured-outputs-in-the-api/)
- OpenAI: [New tools for building agents](https://openai.com/index/new-tools-for-building-agents/)
- OpenAI: [Computer-Using Agent](https://openai.com/index/computer-using-agent/)
- OpenAI: [Unlocking the Codex harness](https://openai.com/index/unlocking-the-codex-harness/)
- Anthropic: [How to implement tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools)
- Anthropic: [Computer use tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool)
- Anthropic: [Introducing MCP](https://www.anthropic.com/news/model-context-protocol)
- Anthropic: [Writing effective tools for AI agents](https://www.anthropic.com/engineering/writing-tools-for-agents)
- Google: [Function calling with Gemini API](https://ai.google.dev/gemini-api/docs/function-calling)
- Google: [Code execution](https://ai.google.dev/gemini-api/docs/code-execution)
- LangChain: [Tools](https://docs.langchain.com/oss/python/langchain-tools)
- LangChain: [LangSmith Observability](https://docs.langchain.com/oss/python/langchain/observability)

### Benchmark / 项目

- [BFCL](https://gorilla.cs.berkeley.edu/leaderboard)
- [τ-bench](https://www.tau-bench.com/)
- [WebArena](https://webarena.dev/)
- [OSWorld](https://arxiv.org/abs/2404.07972)
- [WorkArena](https://arxiv.org/abs/2403.07718)
- [Terminal-Bench](https://www.tbench.ai/)
- [MLE-bench](https://openai.com/index/mle-bench/)
- [SWE-agent](https://arxiv.org/abs/2405.15793)
- [OpenHands](https://github.com/OpenHands/OpenHands)
- [browser-use](https://github.com/browser-use/browser-use)

### 安全 / 事故

- NCSC: [Prompt injection is not SQL injection](https://www.ncsc.gov.uk/blog-post/prompt-injection-is-not-sql-injection)
- OWASP: [Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications)
- OWASP: [MCP Tool Poisoning](https://owasp.org/www-community/attacks/MCP_Tool_Poisoning)
- MCP: [Security Best Practices](https://modelcontextprotocol.io/specification/2025-06-18/basic/security_best_practices)
- Embrace The Red: [Anthropic Claude Data Exfiltration Vulnerability Fixed](https://embracethered.com/blog/posts/2023/anthropic-fixes-claude-data-exfiltration-via-images/)
- [Securing the Model Context Protocol](https://arxiv.org/abs/2512.06556)
- [MCPTox](https://arxiv.org/abs/2508.14925)
