# LLM API 综合调研

调研日期：2026-05-30（Asia/Shanghai）

## 这篇文档回答什么

这篇文档调研的不是“哪个模型最好”，而是：

> 当 LLM 能力主要通过 API 进入软件系统时，API 本身变成了什么样的工程对象？

资料范围覆盖：

- 论文和技术报告。
- 官方文档。
- Benchmark / eval。
- 开源项目。
- 工程博客和生产案例。
- 开发者社区讨论。
- 招聘市场。
- 事故复盘。
- 历史类比。

这篇文档会刻意区分三类内容：

- `事实`：来源直接支持的信息，例如官方文档、论文、benchmark 主页、事故复盘、招聘描述。
- `观点`：来源作者、社区或本文给出的判断，不等同于事实。
- `推断`：基于多类来源做出的综合判断，需要持续用新资料校正。

## 核心结论

1. `事实` LLM API 已经从早期的文本补全接口，演化成包含多轮上下文、工具调用、结构化输出、RAG / 文件检索、批处理、实时音频、多模态、eval 和 agent runtime 的平台接口。
2. `事实` OpenAI、Anthropic、Google Gemini、AWS Bedrock、Mistral 等主流平台都在 API 层支持工具调用或函数调用，结构化 JSON 输出也从“prompt 约定”变成“schema 约束”。
3. `观点` 真正生产可用的 LLM API 集成，不是“调用一次模型”，而是一个小型分布式系统：要处理延迟、限流、重试、幂等、成本、日志、版本、回滚、权限、数据治理和 eval。
4. `事实` OpenAI 的 Responses API 文档明确把 Responses 描述为新的 API primitive，并建议新项目使用 Responses；Chat Completions 仍受支持，但不再是新能力的主要承载面。
5. `事实` Benchmark 正在从“模型答题”扩展到“模型是否能正确选择工具、生成参数、执行多步任务、遵守业务规则”。BFCL、ToolBench、API-Bank、Tau-bench 等都围绕工具/API 使用能力设计。
6. `推断` 未来的差异不会只来自模型参数，而会来自 `model + API surface + tool protocol + runtime + eval + observability + governance` 的组合。
7. `事实` 开源生态已经出现一批 LLM API 基础设施：LiteLLM、Portkey、Helicone、Langfuse、OpenTelemetry GenAI semantic conventions、Instructor、BAML、promptfoo、Ragas 等。
8. `事实` 事故案例显示，LLM API 系统的失败模式包括：模型幻觉被当成公司承诺、敏感数据暴露、供应商服务降级、prompt injection、API key 泄露、成本失控、工具误调用。
9. `推断` 招聘市场正在把“会写 prompt”重新拆成更工程化的能力：eval design、agent harness、tool use、LLMOps、observability、可靠后端、生产数据闭环。
10. `观点` 评价 LLM API 的核心问题不该是“能不能生成答案”，而是“它能不能在有边界、有成本、有审计、有回滚的系统里稳定产生可验收结果”。

## 来源地图

| 范围 | 代表资料 | 可用来回答什么 |
|---|---|---|
| 官方文档 | OpenAI Responses / Tools / Structured Outputs / Evals / Batch；Anthropic Messages / Tool Use / Prompt Caching；Gemini Function Calling / Structured Output / Context Caching；AWS Bedrock Converse / Guardrails | 主流 API surface 正在包含哪些工程原语 |
| 论文 / 技术报告 | ReAct、Toolformer、Gorilla、ToolBench、API-Bank、Tau-bench | LLM 调用工具和 API 的研究脉络 |
| Benchmark / eval | BFCL、ToolBench、API-Bank、Tau-bench、SWE-bench、Terminal-Bench、OpenAI eval best practices | 如何评估 tool calling、agent loop 和真实任务 |
| 开源项目 | LiteLLM、Portkey、Helicone、Langfuse、OpenTelemetry GenAI、Instructor、BAML、promptfoo、Ragas、LangChain / LangGraph、LlamaIndex、Vercel AI SDK | 开发者在生产里补齐哪些 API 基础设施 |
| 工程案例 | Shopify Sidekick、Anthropic Building Effective Agents、OpenAI eval 文档、Stripe Minions 等 | 生产系统如何把模型 API 包在 workflow / harness / eval 里 |
| 社区讨论 | GitHub issues、Reddit、HN、厂商论坛、SDK issue、LLMOps 工具社区 | 开发者实际遇到的痛点和非正式经验 |
| 招聘市场 | OpenAI Applied Evals、Anthropic Model Evaluations、Agent Prompts & Evals 等岗位 | 企业正在为哪些 LLM API 能力付薪水 |
| 事故复盘 | OpenAI Redis bug、Anthropic 近期服务问题复盘、Air Canada chatbot 案例、DPD chatbot 事件等 | LLM API 系统的真实失败模式 |
| 历史类比 | 云 API、支付 API、搜索 API、数据库驱动、消息队列、浏览器自动化 | 哪些软件工程经验可迁移，哪些不能直接套用 |

## 什么是 LLM API

`事实` 最窄的定义：LLM API 是把模型能力通过 HTTP / SDK 暴露给应用的接口，输入通常是文本、图片、音频、文件或对话消息，输出是文本、结构化数据、工具调用、图片、音频或中间事件。

`事实` 更贴近 2026 年生产实践的定义：LLM API 不再只是 `prompt -> completion`，而是包括这些能力：

| API 原语 | 作用 |
|---|---|
| 文本 / 多模态生成 | 生成文本、图片描述、音频、代码、摘要、分类等 |
| 多轮上下文 | 显式传入历史消息，或用 response / conversation / thread id 管理状态 |
| 流式输出 | 用 SSE / WebSocket 等方式逐步返回 token、事件、工具调用和状态 |
| 工具调用 / 函数调用 | 让模型选择工具，并生成符合 schema 的参数 |
| 结构化输出 | 用 JSON Schema / 类型系统约束模型最终输出 |
| 检索 / 文件搜索 | 把外部文档、知识库、文件作为上下文输入 |
| 上下文缓存 | 对长前缀、系统上下文、文档上下文做缓存以降低成本和延迟 |
| 批处理 | 对离线任务、eval、分类、embedding 等进行异步批量调用 |
| Realtime | 支持低延迟语音、音频、流式多模态交互 |
| Agent runtime | 把模型、工具、状态、审批、handoff、trace 组合成 agent 工作流 |
| Guardrails / Moderation | 对输入、输出、工具调用和高风险场景加安全限制 |
| Evals | 对 prompt、模型、工具和系统整体做可重复评估 |

`观点` LLM API 的核心变化是：API 的返回值不再只是“答案”，还可能是“下一步动作”。这使它更像一个决策接口，而不是传统 RPC。

`推断` 这个变化会让 LLM API 的工程难度接近“消息队列 + 工作流引擎 + 外部 SaaS API + 不确定性模型”的组合。

## API 形态演进

### 从 completion 到 conversation

`事实` 早期 GPT-3 API 更像通用文本补全。开发者把 prompt 拼成一段文本，再解析模型续写内容。

`事实` Chat Completions 把输入升级为角色消息列表，例如 system、user、assistant，并成为一段时间内主流 LLM 应用的基础接口。

`推断` 消息接口的意义不只是格式更整齐，而是把“谁说了什么”“哪些内容有更高优先级”“哪些历史应该保留”变成 API 层的一部分。

### 从 conversation 到 tool-using API

`事实` 2023 年后，OpenAI function calling、Anthropic tool use、Gemini function calling、Mistral function calling、AWS Bedrock tool use 等能力逐渐成为主流 API 功能。

`事实` 这些接口通常要求开发者声明工具名、描述、参数 schema，模型返回工具调用请求，应用再执行工具并把结果返回给模型。

`观点` 这一步是 LLM API 最重要的工程转折：模型开始参与控制流，而不仅是生成自然语言。

### 从 prompt contract 到 schema contract

`事实` OpenAI Structured Outputs 文档说明，Structured Outputs 可以让模型输出遵循开发者提供的 JSON Schema；相比 JSON mode，只保证有效 JSON 不等于保证 schema 正确。

`事实` Gemini、Mistral、Anthropic 等平台也提供结构化输出、JSON mode 或类似能力。

`观点` 结构化输出是把 LLM 接入软件系统的基本卫生条件。没有 schema、校验和错误分支，LLM 输出很难成为可靠的下游输入。

### 从单次调用到 agent runtime

`事实` OpenAI Responses API 文档把 Responses 描述为统一接口，支持内置工具、函数、自定义工具、MCP、文件搜索、web search、computer use、code interpreter、多轮状态等 agentic primitives。

`事实` Anthropic 的 Building Effective Agents 文章强调：优先使用简单 workflow，只有任务确实需要动态决策时才使用 agent。

`推断` 主流 API 厂商会继续把更多 agent runtime 能力上移到平台层，但企业仍需要自己的 harness 来处理权限、数据、业务工具、审计和回滚。

## 主流平台 API 对比

### OpenAI

`事实` OpenAI 当前文档建议新项目使用 Responses API；Chat Completions 仍然受支持。

`事实` Responses API 的重要特征包括：

- `input` / `instructions` 语义比 Chat Completions 更清晰。
- 返回 typed `output` items，而不是只有 message。
- 支持 `previous_response_id` 或 Conversations API 处理多轮状态。
- 支持 web search、file search、computer use、code interpreter、image generation、remote MCP、自定义 function 等工具。
- Structured Outputs 在 Responses 中使用 `text.format`。
- Evals 文档建议构建持续评估流程，而不是靠感觉判断效果。
- Batch API 适合离线任务、eval、大规模分类等，官方文档说明相比同步 API 有成本折扣和更高批处理 headroom。
- Prompt caching cookbook 把 `prompt_cache_key` 解释成影响缓存命中的路由提示，并提示缓存是 best-effort。

`观点` OpenAI API 的方向是把“模型调用”升级成“agentic application platform”。对开发者来说，API 变简单的同时，平台语义也变重：状态、存储、工具、数据保留、trace、审批都需要认真设计。

### Anthropic

`事实` Anthropic Messages API 以 message 列表、system prompt、max tokens、tools 等参数为核心；Claude 文档提供 tool use、prompt caching、batch、rate limits、computer use、MCP 等能力说明。

`事实` Anthropic 的 Building Effective Agents 把 LLM 系统区分为 workflow 和 agent：workflow 是预定义路径，agent 是模型动态决定流程。

`观点` Anthropic 的公开材料特别强调“保持简单”“工具设计清晰”“先从可控 workflow 开始”。这对 API 工程很重要，因为很多问题不该一开始就交给全自主 agent。

### Google Gemini API

`事实` Gemini API 文档覆盖 function calling、structured output、context caching、batch mode、code execution、grounding、multimodal 输入输出等能力。

`事实` Gemini 的长上下文和多模态能力在官方材料中经常是重点。

`推断` Gemini API 更适合纳入需要长上下文、多模态、Google 生态集成的系统，但生产设计仍要验证具体模型、区域、速率、价格和工具支持。

### AWS Bedrock / Azure OpenAI / Vertex AI

`事实` AWS Bedrock Converse API 提供跨模型的对话接口，并支持 tool use、guardrails、模型选择和企业云治理能力。

`事实` Azure OpenAI 和 Vertex AI 更强调云账号、网络、权限、审计、区域、合规、企业身份集成和统一云账单。

`观点` 企业采购 LLM API 时，模型能力只是一个维度。数据驻留、审计、私网连接、组织权限、合规承诺、SLA 和现有云栈集成往往会决定真实选择。

### Mistral、Cohere、开源模型服务商

`事实` Mistral API 文档也支持 function calling、JSON / structured output、batch inference、agents 等能力。

`事实` Cohere、Together、Groq、Fireworks、DeepInfra、OpenRouter 等提供不同形态的 hosted model API 或聚合 API。

`推断` 当底层生成接口越来越接近 OpenAI-compatible 时，竞争会转向价格、延迟、吞吐、模型选择、企业治理、工具生态和稳定性。

## 论文和技术报告脉络

| 资料 | 类型 | 事实 | 对 LLM API 的启发 |
|---|---|---|---|
| ReAct | 论文 | 把 reasoning trace 和 acting 结合，让模型在推理和外部动作之间交替 | API 不能只看最终答案，要记录轨迹 |
| Toolformer | 论文 | 探索模型如何学习在文本中插入工具调用 | 工具调用可被模型学习，但工具边界仍要工程化 |
| Gorilla | 论文 / 项目 | 关注模型如何根据自然语言调用大量 API | API schema、文档检索和参数生成是独立能力 |
| API-Bank | Benchmark | 用一组模拟 API 和对话任务评估 tool-augmented LLM | 工具调用要评估多轮状态和真实执行结果 |
| ToolBench / ToolLLM | Benchmark / 框架 | 使用大规模真实 API 构造工具使用任务 | 真实 API 数量大时，需要检索、规划、纠错 |
| BFCL | Benchmark | 评估函数调用、参数生成、多工具、并行调用等能力 | tool calling 是独立于聊天偏好的能力维度 |
| Tau-bench | Benchmark | 用有状态业务环境评估 agent 是否遵守规则并正确调用 API | LLM API 要放进业务状态机里评估 |

`观点` 这些研究共同说明：LLM API 的难点不是“会不会输出 JSON”，而是能否在给定 API 文档、业务规则、状态变化和失败反馈时做出正确动作。

`推断` 未来更重要的 eval 会越来越像“业务流程模拟器”，而不是静态问答题。

## Benchmark / Eval 怎么看

### Function calling benchmark

`事实` BFCL 关注函数选择、参数抽取、并行函数、多轮调用、实时更新函数文档等能力。

`事实` ToolBench、API-Bank、Gorilla 等工作把工具/API 调用本身当成模型能力来评测。

`观点` Function calling benchmark 适合筛选模型和 API 表达能力，但不能单独证明一个生产 agent 可用，因为生产系统还包含权限、工具可靠性、业务数据、回滚、用户体验和成本。

### Agent / workflow benchmark

`事实` SWE-bench、Terminal-Bench、Tau-bench、MLE-bench 等更接近多步任务：模型需要读上下文、调用工具、修改环境、处理反馈。

`观点` 对 API 应用来说，最有价值的指标通常不是单次 accuracy，而是：

| 指标 | 含义 |
|---|---|
| `task_success_rate` | 最终任务是否完成 |
| `tool_call_accuracy` | 工具是否选对、参数是否正确 |
| `schema_valid_rate` | 输出是否满足 schema |
| `policy_compliance_rate` | 是否遵守业务规则和权限 |
| `cost_per_success` | 每个成功任务的平均成本 |
| `latency_p50 / p95` | 用户可感知延迟 |
| `retry_recovery_rate` | 遇到工具错误或模型错误后能否恢复 |
| `human_escalation_rate` | 需要人工介入的比例 |
| `regression_rate` | prompt、模型或工具升级后老任务失败比例 |

### 自建 eval 的必要性

`事实` OpenAI eval best practices 文档强调，生成式 AI 有变异性，传统测试不足以覆盖，需要任务特定 eval，并建议持续评估、日志沉淀、人工反馈校准、自动评分结合。

`观点` LLM API 系统没有自建 eval，就相当于没有回归测试。换模型、改 prompt、改工具 schema、改检索策略，都可能引入行为回归。

`推断` 长期看，企业会为每条关键 LLM API workflow 建立小型 benchmark：几十到几百条生产样本、边界样本、攻击样本、失败样本，并接入 CI 或发布门禁。

## 开源生态

### API 网关和模型路由

| 项目 | 类型 | 解决的问题 |
|---|---|---|
| LiteLLM | OpenAI-compatible 代理 / SDK | 统一多供应商 API、路由、fallback、budget、virtual keys |
| Portkey AI Gateway | LLM gateway | 路由、重试、缓存、guardrails、日志、治理 |
| OpenRouter | 聚合服务 | 多模型访问和统一账单 |
| Vercel AI SDK | 应用 SDK | 前端 / 全栈应用中的流式 UI、provider 抽象、tool calling |

`观点` LLM gateway 的出现说明，开发者已经把模型 API 当成可路由、可观测、可治理的外部依赖，而不是写死在业务代码里的 SDK 调用。

### Observability 和 eval

| 项目 | 类型 | 解决的问题 |
|---|---|---|
| Langfuse | 开源 LLM observability / prompt management / eval | trace、session、cost、prompt 版本、dataset、eval |
| Helicone | LLM observability proxy | 请求日志、成本、延迟、用户维度、缓存 |
| OpenTelemetry GenAI semantic conventions | 标准 | 给 LLM 请求、prompt、completion、token、模型、工具调用定义遥测字段 |
| promptfoo | eval / red-team | prompt 和模型回归测试、安全测试 |
| Ragas | RAG eval | 检索增强问答的 context precision、faithfulness 等 |
| Arize Phoenix | observability / eval | tracing、RAG 分析、LLM eval |

`推断` LLM API observability 会从“记录 prompt 和 response”升级成“记录完整 trajectory”：输入、检索、模型调用、工具调用、schema 校验、重试、成本、用户反馈、最终业务结果。

### 类型安全和结构化输出

| 项目 | 类型 | 解决的问题 |
|---|---|---|
| Instructor | Python / TS structured output | 用 Pydantic / schema 驱动 LLM 输出解析和重试 |
| BAML | DSL / runtime | 将 LLM 调用、prompt、schema、测试组织成可维护接口 |
| Zod / Pydantic SDK helpers | 类型工具 | 减少 JSON Schema 与代码类型漂移 |

`观点` 如果一个 LLM API 调用会进入业务流程，就应该被当成一个 typed function，而不是任意字符串生成。

### Agent runtime / workflow

| 项目 | 类型 | 解决的问题 |
|---|---|---|
| LangGraph | Agent workflow / durable execution | 状态图、checkpoint、人类介入、多 agent |
| LlamaIndex | RAG / agent data framework | 数据连接、索引、检索、agent tools |
| Semantic Kernel | Agent / planner / enterprise integration | .NET / Python 生态中的插件和 planner |
| AutoGen / CrewAI | Multi-agent framework | 多 agent 协作、角色、任务编排 |

`观点` 这类框架的价值不是“让 agent 更神奇”，而是把上下文、状态、工具、重试和 trace 显式化。

## 开发者社区讨论信号

`事实` GitHub issues、Reddit、HN、厂商论坛和 SDK issue 中反复出现以下问题类型：

- 不同供应商都声称 OpenAI-compatible，但 streaming event、tool calling、JSON mode、错误码、token 统计、模型参数并不完全一致。
- 模型升级后，同一 prompt 的格式、语气、工具选择、拒答率和成本可能变化。
- Function calling 不是强类型 RPC；模型仍可能选择错误工具、遗漏字段、填错 enum、过度调用工具。
- 流式输出和工具调用结合后，前端状态机复杂度明显上升。
- 长上下文并不等于高有效上下文；检索、排序、去噪、压缩仍然重要。
- Rate limit、context limit、output token limit、超时、队列延迟会直接影响产品体验。
- API key 泄露和前端直连是常见安全错误。
- “先上线再补 eval”会导致后续每次改 prompt 都像赌博。

`观点` 社区讨论最有价值的部分不是单个抱怨，而是暴露了 LLM API 的真实边界：它不是确定性库函数，而是一个需要观测、控制和回归测试的外部智能服务。

`推断` 团队越早把 LLM API 抽象成内部平台能力，越容易在模型替换、成本优化、故障切换和审计上获得回旋空间。

## 工程案例

### Anthropic：Building Effective Agents

`事实` Anthropic 建议优先使用简单、可组合的 workflow；当任务需要模型动态决策时，再使用 agent。

`观点` 这对 LLM API 工程很关键：确定性控制流、业务规则、权限审批、数据写入不应轻易交给模型自由决定。

### Shopify：生产级 agentic systems

`事实` Shopify 工程文章围绕生产级 agentic systems 讨论了工具、上下文、评估、可靠性和产品集成。

`观点` 这类案例说明，LLM API 的生产难点主要在“把模型嵌入真实业务系统”，而不在 demo 阶段的聊天体验。

### OpenAI：Evals 和 Responses

`事实` OpenAI 官方 eval 文档把 eval 描述为处理生成式 AI 变异性的结构化测试，并建议建立持续评估。

`事实` OpenAI Responses API 把 built-in tools、function tools、MCP、状态和 typed items 放到统一接口里。

`推断` OpenAI 正在把 API 从“模型访问层”推进到“agent 应用平台层”。

### Stripe / coding agent 案例

`事实` Stripe Minions 等工程案例显示，复杂 agent 系统通常混合 deterministic workflow 和 agent nodes：确定性代码负责测试、git、CI、状态推进，模型负责开放性理解和修改。

`观点` 这也是一般 LLM API 应用的原则：能用普通代码可靠完成的部分，不要交给模型；模型应处理语义理解、模糊决策和自然语言交互。

## 招聘市场信号

`事实` OpenAI 的 Applied Evals 岗位描述强调设计 agents、harnesses、eval pipelines，把真实工作流转成可复现质量信号，并连接产品质量和训练反馈。

`事实` Anthropic 的 Model Evaluations 岗位强调能力评估、分布式 eval 执行平台、训练期间模型健康 dashboard、prompting / sampling / scaffolding 对结果的影响，以及观测和实验设计。

`事实` 2025 年关于 prompt engineer 职位的调研论文分析了 LinkedIn 上的 AI 岗位样本，说明 prompt engineer 已经从媒体热词进入更具体的技能组合讨论。

`推断` 招聘市场正在把 LLM API 相关能力拆成几类岗位：

| 能力 | 对应岗位信号 |
|---|---|
| API / 平台工程 | LLM gateway、model routing、quota、billing、auth、observability |
| Evals 工程 | dataset、grader、human review、regression、continuous evaluation |
| Agent / harness 工程 | tool use、runtime、state、permission、workflow、handoff |
| Applied AI / product engineering | 把模型能力转成真实业务流程和用户体验 |
| Prompt / context engineering | 系统 prompt、上下文组织、工具说明、模型行为调试 |
| Safety / policy / red team | jailbreak、prompt injection、policy compliance、risk review |

`观点` 单纯“会调 prompt”的价值会下降；能把 prompt、API、工具、eval、日志、数据治理和产品闭环一起做出来的工程师价值会上升。

## 事故复盘

### 模型幻觉被当成公司承诺：Air Canada

`事实` 2024 年加拿大 Civil Resolution Tribunal 的 Moffatt v. Air Canada 案例中，Air Canada 网站 chatbot 给出了错误的 bereavement fare 信息，裁决认为公司需要对 chatbot 提供的信息承担责任。

`观点` 这是 LLM API 产品的基本警示：如果用户把输出当成正式承诺，系统就必须有来源、边界、免责声明、人工升级和政策一致性测试。

### 数据暴露和依赖问题：OpenAI Redis bug

`事实` OpenAI 2023 年 3 月 ChatGPT 事故复盘提到，开源 Redis client bug 触发了部分用户聊天标题和部分支付相关信息暴露。

`观点` 这不是“模型幻觉”问题，而是常规软件供应链和分布式系统问题。LLM API 产品仍然会被普通工程 bug 击穿。

### 供应商服务问题：Anthropic postmortem

`事实` Anthropic 2025 年 9 月工程复盘总结了三个近期基础设施问题，这些问题导致 Claude 响应质量间歇性下降，并促使团队改进检测、评估和内部流程。

`推断` 对依赖外部 LLM API 的应用来说，供应商可靠性必须进入架构：超时、fallback、降级答案、排队、任务恢复、用户告知都要设计。

### Prompt injection 和品牌风险：DPD / dealership chatbot

`事实` 多起公开报道显示，接入聊天机器人的企业遇到过 prompt injection、越权输出、辱骂、承诺异常价格等品牌风险事件。

`观点` 这些案例通常不适合作为“LLM 必然危险”的证据，但适合作为产品边界设计案例：用户输入不能直接改变系统身份、政策、价格、合同承诺或工具权限。

### API key 泄露和成本失控

`事实` 开发者社区长期存在把 API key 放进前端、日志、仓库、notebook、截图或 CI 输出导致泄露的案例。

`观点` LLM API 的 token 计费让泄露风险不只是数据风险，也是直接财务风险。必须使用后端代理、最小权限、额度、告警、key rotation 和环境隔离。

## 历史类比

### 像云 API

`推断` LLM API 像 AWS EC2 / S3 的地方：

- 把稀缺基础设施能力以 API 形式开放。
- 价格、限额、区域、SLA、配额和账单成为架构参数。
- 生态会围绕 SDK、网关、监控、成本优化、安全治理形成。

不像的地方：

- 云 API 多数是确定性资源操作，LLM API 输出有不确定性。
- 云 API 的回归主要来自系统和配置，LLM API 的回归还来自模型行为变化。

### 像支付 API

`推断` LLM API 像 Stripe / 支付 API 的地方：

- API 调用会触发真实业务后果。
- 需要幂等、审计、风控、权限、回滚和人工介入。
- 错误处理比 happy path 更重要。

不像的地方：

- 支付 API 的状态机通常由平台严格定义；LLM API 可能由模型在运行时选择工具和路径。

### 像搜索 API

`推断` LLM API 像搜索 API 的地方：

- 输出质量依赖召回、排序、上下文和用户意图理解。
- 结果可能不完整，需要展示来源和置信边界。

不像的地方：

- 搜索 API 返回候选结果，LLM API 常直接生成自然语言结论，用户更容易把它当成确定事实。

### 像数据库驱动

`推断` LLM API 像数据库驱动的地方：

- 应该被封装成内部接口，而不是散落在业务代码里。
- 需要连接管理、超时、重试、观测、迁移和版本管理。

不像的地方：

- 数据库查询的 schema 稳定得多；LLM 的行为 schema 需要 eval 和监控持续维护。

## 生产架构参考

一个更稳妥的 LLM API 架构可以分成这些层：

```text
用户请求
  -> 产品入口 / 权限检查
  -> prompt & context builder
  -> retrieval / file search / memory
  -> LLM gateway / model router
  -> model API call
  -> tool call dispatcher
  -> output validator / schema parser
  -> policy guardrail / human escalation
  -> business action / final response
  -> trace / eval / feedback / cost accounting
```

`观点` 不要让业务代码到处直接调用 provider SDK。至少应该有一个内部 wrapper，统一处理：

- provider 和模型选择。
- 超时、重试、fallback。
- token 和成本记录。
- prompt 版本。
- schema 校验。
- tool call 审批。
- trace id。
- 用户 / 租户 / 任务 metadata。
- 错误分类。
- eval 数据采样。

## 设计清单

### API 选择

- `事实` 确认目标模型是否支持需要的能力：tool calling、structured output、vision、audio、long context、prompt caching、batch、region、ZDR / data retention。
- `观点` 不要只看“模型强”，要评估 latency、cost、rate limit、SDK 成熟度、错误语义、供应商状态页、企业合规和迁移成本。
- `推断` 对关键业务，至少保留一个可降级路径：较弱模型、模板答案、人工处理、离线任务或只读模式。

### Prompt 和上下文

- `事实` 长上下文会带来成本、延迟和注意力稀释；上下文缓存只能优化部分场景。
- `观点` prompt 应该像代码一样版本化，重要 prompt 应该有测试。
- `推断` 最稳的上下文策略通常是“少而准”：明确任务、规则、可用工具、输出 schema、拒绝条件和少量关键示例。

### 工具调用

- `事实` 主流 API 都让开发者用 schema 描述工具参数。
- `观点` 工具应该是小而明确的业务能力，不要给模型一个万能 `execute_sql`、`run_command` 或 `call_internal_api`。
- `推断` 高风险工具应该做二次确认、权限检查、dry run、人类审批或环境隔离。

### 结构化输出

- `事实` JSON mode 不等于 schema adherence；Structured Outputs / schema-based output 更适合下游系统。
- `观点` 所有进入数据库、API、队列、工作流状态机的模型输出都应该经过 parser、validator 和错误分支。
- `推断` 如果 schema 很复杂，应该拆成多个小步骤，而不是要求模型一次输出一个巨大对象。

### 可靠性

- `事实` LLM API 会遇到限流、超时、供应商降级、模型变更、上下文过长、输出截断。
- `观点` 重试不能无脑做；要区分 rate limit、server error、schema invalid、policy refusal、tool error、用户输入不足。
- `推断` 对长任务，应把每一步状态持久化，支持 resume，而不是依赖一次长调用。

### 安全和数据治理

- `事实` LLM API 请求可能包含用户数据、内部文档、商业秘密、代码、日志、截图和工具结果。
- `观点` 敏感数据进入模型前应做分类、脱敏、最小化和供应商政策检查。
- `推断` 关键系统需要记录“哪些数据给了哪个模型、哪个版本、用于什么目的、输出被谁使用、是否触发动作”。

### Eval 和发布

- `事实` OpenAI 等官方文档建议构建任务特定 eval，并持续评估。
- `观点` 模型、prompt、工具 schema、检索策略、系统消息、温度、reasoning effort 都应该视为可引发行文回归的变更。
- `推断` LLM API 发布门禁至少应该包括：golden set、失败样本、adversarial cases、成本预算、延迟预算、人工抽样 review。

### 成本

- `事实` 成本由输入 token、输出 token、reasoning token、缓存命中、批处理、重试、工具调用、并发和模型选择共同决定。
- `观点` 成本指标应该按 `cost_per_success` 和 `human_minutes_saved` 看，而不只是每百万 token 价格。
- `推断` 真正的成本优化通常来自：少放上下文、减少重试、缓存长前缀、使用小模型分流、批处理离线任务、避免把简单规则交给大模型。

## 反模式

- `观点` 把 LLM API 当成普通函数调用，不记录 prompt、模型版本、输入、输出和成本。
- `观点` 把 API key 放在浏览器、移动端或公开仓库。
- `观点` 让模型直接决定付款、退款、删除、部署、发邮件、改权限等高风险动作。
- `观点` 用“请严格输出 JSON”替代 schema、parser、validator 和 retry。
- `观点` 没有 eval 就换模型或改 prompt。
- `观点` 把供应商 benchmark 当成自己业务上线标准。
- `观点` 把用户输入、检索文档和工具输出混在一个 prompt 里，不做来源和优先级隔离。
- `观点` 只优化 demo 的成功路径，不设计拒答、超时、限流、升级人工和审计。

## 分层判断框架

可以用下面这张表判断一个 LLM API 集成是否成熟：

| 层级 | 特征 | 风险 |
|---|---|---|
| Level 0：裸调用 | 业务代码直接调用 SDK，prompt 写在代码里 | 难回归、难审计、难迁移 |
| Level 1：封装调用 | 有内部 wrapper、基本日志、错误处理 | 能用，但行为质量仍靠人工感觉 |
| Level 2：结构化接口 | schema 输出、tool schema、prompt 版本、成本记录 | 可以接入部分业务流程 |
| Level 3：可观测系统 | trace、eval dataset、dashboard、fallback、限额、告警 | 可以进入生产，但需持续维护 |
| Level 4：治理平台 | 多模型路由、权限、审计、数据策略、发布门禁、人类反馈闭环 | 适合多团队复用 |
| Level 5：自进化闭环 | 生产样本 -> eval -> prompt / tool / model 改进 -> 安全发布 | 高投入，只适合高价值场景 |

`推断` 大多数团队不需要一开始做到 Level 5，但至少应该尽快从 Level 0 升到 Level 2。否则后面每次模型和 prompt 变化都会变成不可控风险。

## 后续值得继续跟踪的问题

- OpenAI Responses API、Anthropic Messages API、Gemini API、Bedrock Converse API 的 tool calling 语义会不会进一步收敛？
- MCP 是否会成为跨模型工具接入的事实标准，还是只在 agent / IDE / enterprise integration 场景里流行？
- Structured Outputs 能否覆盖更复杂 schema，还是开发者仍需要外部 validator 和 retry 框架？
- LLM API 的流式事件格式是否会标准化？
- OpenTelemetry GenAI semantic conventions 能否成为 LLM observability 的共同字段标准？
- OpenAI-compatible 接口会在多大程度上削弱 provider lock-in？
- 生产 eval 会不会成为所有 AI 产品团队的默认 CI？
- 招聘市场会把 prompt engineer 吸收到 applied AI engineer、eval engineer、agent engineer、LLMOps engineer 这些更明确的岗位里吗？

## 参考资料

官方文档：

- OpenAI: [Migrate to the Responses API](https://developers.openai.com/api/docs/guides/migrate-to-responses)
- OpenAI: [Using tools](https://developers.openai.com/api/docs/guides/tools)
- OpenAI: [Structured model outputs](https://developers.openai.com/api/docs/guides/structured-outputs)
- OpenAI: [Evaluation best practices](https://developers.openai.com/api/docs/guides/evaluation-best-practices)
- OpenAI: [Batch API](https://developers.openai.com/api/docs/guides/batch)
- OpenAI Cookbook: [Prompt Caching 201](https://developers.openai.com/cookbook/examples/prompt_caching_201)
- Anthropic: [Messages API](https://docs.anthropic.com/en/api/messages)
- Anthropic: [Tool use](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview)
- Anthropic: [Building effective agents](https://www.anthropic.com/research/building-effective-agents)
- Google Gemini API: [Function calling](https://ai.google.dev/gemini-api/docs/function-calling)
- Google Gemini API: [Structured output](https://ai.google.dev/gemini-api/docs/structured-output)
- Google Gemini API: [Context caching](https://ai.google.dev/gemini-api/docs/caching)
- Google Gemini API: [Batch mode](https://ai.google.dev/gemini-api/docs/batch-mode)
- AWS Bedrock: [Converse API](https://docs.aws.amazon.com/bedrock/latest/userguide/conversation-inference.html)
- AWS Bedrock: [Tool use](https://docs.aws.amazon.com/bedrock/latest/userguide/tool-use.html)
- AWS Bedrock: [Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html)
- Mistral AI: [Function calling](https://docs.mistral.ai/capabilities/function_calling/)
- Mistral AI: [Structured outputs](https://docs.mistral.ai/capabilities/structured_output/)

论文、benchmark 和技术报告：

- ReAct: [Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)
- Toolformer: [Language Models Can Teach Themselves to Use Tools](https://arxiv.org/abs/2302.04761)
- Gorilla: [Large Language Model Connected with Massive APIs](https://arxiv.org/abs/2305.15334)
- API-Bank: [A Comprehensive Benchmark for Tool-Augmented LLMs](https://arxiv.org/abs/2304.08244)
- ToolBench: [An Open Platform for Training, Serving, and Evaluating Large Language Models for Tool Learning](https://arxiv.org/abs/2307.16789)
- Berkeley Function Calling Leaderboard: [BFCL](https://gorilla.cs.berkeley.edu/leaderboard.html)
- Tau-bench: [A Benchmark for Tool-Agent-User Interaction in Real-World Domains](https://arxiv.org/abs/2406.12045)
- SWE-bench: [Official site](https://www.swebench.com/)
- Terminal-Bench: [Official site](https://terminal-bench.com/)

开源项目：

- [LiteLLM](https://github.com/BerriAI/litellm)
- [Portkey AI Gateway](https://github.com/Portkey-AI/gateway)
- [Langfuse](https://github.com/langfuse/langfuse)
- [Helicone](https://github.com/Helicone/helicone)
- [OpenTelemetry GenAI semantic conventions](https://github.com/open-telemetry/semantic-conventions/tree/main/docs/gen-ai)
- [Instructor](https://github.com/instructor-ai/instructor)
- [BAML](https://github.com/BoundaryML/baml)
- [promptfoo](https://github.com/promptfoo/promptfoo)
- [Ragas](https://github.com/explodinggradients/ragas)
- [LangGraph](https://github.com/langchain-ai/langgraph)
- [LlamaIndex](https://github.com/run-llama/llama_index)
- [Vercel AI SDK](https://github.com/vercel/ai)

工程案例、招聘和事故：

- Shopify Engineering: [Building production-ready agentic systems](https://shopify.engineering/building-production-ready-agentic-systems)
- Anthropic Engineering: [A postmortem of three recent issues](https://www.anthropic.com/engineering/a-postmortem-of-three-recent-issues)
- OpenAI: [March 20 ChatGPT outage](https://openai.com/index/march-20-chatgpt-outage/)
- Civil Resolution Tribunal: [Moffatt v. Air Canada, 2024 BCCRT 149](https://decisions.civilresolutionbc.ca/crt/crtd/en/item/525448/index.do)
- OpenAI Careers: [Software Engineer, Applied Evals](https://openai.com/careers/software-engineer-applied-evals/)
- Anthropic Careers: [Research Engineer, Model Evaluations](https://job-boards.greenhouse.io/anthropic/jobs/5198255008)
- arXiv: [Prompt Engineer: Analyzing Hard and Soft Skill Requirements in the AI Job Market](https://arxiv.org/abs/2506.00058)
