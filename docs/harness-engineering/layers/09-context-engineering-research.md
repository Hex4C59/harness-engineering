# Context Engineering 调研

调研日期：2026-05-30

## 这篇文档回答什么

这篇文档围绕 **Context Engineering** 做一次横向调研，资料范围包括论文、博客、技术报告、官方文档、benchmark / eval、开源项目、开发者社区讨论、工程案例、招聘市场、事故复盘和历史类比。

这里的 context engineering 不是“写更长 prompt”，也不是单纯 RAG。它更接近 agent harness 里的一个核心子系统：

```text
在每一步模型调用前，决定模型应该看到什么、不该看到什么、
以什么结构看到、从哪里取、保留多久、如何验证它是否真的有用。
```

本文刻意区分三类陈述：

- **事实**：来源明确说明或可公开验证的内容。
- **观点**：来源作者或社区给出的判断、经验总结、立场。
- **推断**：本文基于多份资料做出的工程归纳，不当作来源原意。

核心结论：

> Context engineering 正在从“prompt engineering 的新名字”演化为 agent 系统的上下文治理工程。它关注的不只是提示词措辞，而是 instructions、tools、retrieval、memory、tool outputs、state、trace、permissions、eval 和 human feedback 如何被组织进有限的 context window。对 coding agent 和长程 agent 来说，它已经是 harness engineering 的关键组成部分。

## 一句话定义

**事实**：[Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) 在 2025-09-29 的工程文章中把 context engineering 描述为一组在 LLM 推理期间维护最佳 token 集合的策略，范围包括 prompt 之外可能进入上下文窗口的系统指令、工具、MCP、外部数据和消息历史。[LangChain](https://www.langchain.com/blog/context-engineering-for-agents) 在 2025-07-02 的文章中把它概括为：在 agent trajectory 的每一步，把恰好合适的信息放进 context window。

**观点**：Andrej Karpathy 的社区表述被 LangChain 引用为：context engineering 是“给下一步填充正确上下文窗口的艺术与科学”。[Cognition](https://cognition.ai/blog/dont-build-multi-agents) 在 2025-06-12 的文章中更强烈地说，context engineering 几乎是构建 AI agent 的第一工作。

**推断**：如果 prompt engineering 是“如何表达任务”，context engineering 就是“如何构造模型的工作记忆”。它把 prompt 从一段文本升级为一个动态组装的数据结构。

## 它和相关概念的边界

| 概念 | 关注点 | 和 Context Engineering 的关系 |
|---|---|---|
| Prompt engineering | 指令怎么写、例子怎么给、输出格式怎么约束 | context engineering 包含 prompt，但不止 prompt |
| RAG | 从外部知识库检索相关内容 | RAG 是 context selection 的一种实现 |
| Memory | 跨 turn、跨 session 保存事实、偏好、计划、经验 | memory 是 context write / select 的来源之一 |
| Tool calling | 模型能调用哪些工具，以及工具输出如何返回 | tool schema 和 tool output 都会占用 context |
| Harness engineering | agent 的运行时、工具、权限、状态、验证和反馈闭环 | context engineering 是 harness 里的上下文治理层 |
| Agent runtime | agent loop、线程、事件、sandbox、持久化 | runtime 执行 context engineering 的策略 |
| Eval | 评估模型或 agent 配置是否有效 | context engineering 必须用 eval 验证，而不是凭感觉 |

**推断**：在本项目的概念体系里，context engineering 应该放在 harness engineering 之下：

```text
Harness Engineering
  = context engineering
  + tool / action surface
  + permissions / sandbox
  + verification loop
  + observability
  + lifecycle / state
  + human handoff
```

换句话说，上下文是 harness 的一部分；harness 不只是上下文。

## 为什么它现在变重要

### 1. Agent 让上下文从静态 prompt 变成动态状态

**事实**：Anthropic 指出，agent 在 loop 里会持续产生可能影响下一轮推理的数据，context 必须被循环地精炼。LangChain 也强调，agent 会交替进行 LLM 调用和工具调用，工具反馈会在多轮中累积，导致 token 数增长、成本和延迟增加，并可能降低 agent 表现。

**推断**：一次普通聊天的上下文主要是 conversation history；一次 coding agent 任务的上下文则包括：

- 用户需求。
- 系统 / 开发者指令。
- `AGENTS.md`、`CLAUDE.md` 等项目规则。
- README、架构文档、设计决策。
- 代码片段和搜索结果。
- shell 命令、测试、lint、typecheck 输出。
- 工具定义和工具调用结果。
- 当前计划、TODO、失败记录。
- diff、review 评论、CI 状态。
- 历史记忆、偏好、项目状态。

这已经不是“写一个好 prompt”能解决的问题。

### 2. 更大的 context window 没有消灭上下文问题

**事实**：《[Lost in the Middle](https://arxiv.org/abs/2307.03172)》研究长上下文模型在多文档问答和 key-value retrieval 中如何使用输入上下文，发现相关信息位于上下文中间时性能会明显下降。

**事实**：LongBench、LongBench v2、RULER、Needle-in-a-Haystack 等 benchmark 都在评估模型是否真的能使用长上下文，而不只是声明支持更大的窗口。

**观点**：Anthropic 的判断是，context 应被视为带有边际收益递减的有限资源；哪怕模型窗口变大，context pollution 和信息相关性问题仍然会存在。

**推断**：长窗口更像更大的内存条，不是自动的数据结构。把所有东西都塞进去，常见失败不是“放不下”，而是：

- 关键事实被噪声淹没。
- 旧错误进入后续推理。
- 工具输出太长，挤掉真正有价值的上下文。
- 多个来源互相冲突，模型没有 provenance 判断能力。
- 成本和延迟失控。

### 3. 工具、MCP 和 agent skills 把“上下文入口”变多了

**事实**：[Model Context Protocol](https://modelcontextprotocol.io/docs/getting-started/intro) 是一个让 AI 应用连接外部系统的开源标准。MCP 官方文档说明，MCP server 可以向 AI 应用提供 data sources、tools 和 prompts；[架构文档](https://modelcontextprotocol.io/docs/learn/architecture) 进一步说明 MCP primitives 包括 `tools`、`resources` 和 `prompts`。

**事实**：MCP 文档同时强调，MCP 只关注 context exchange 协议，不规定 AI 应用如何使用 LLM 或管理提供的上下文。

**推断**：这正好说明 context engineering 的位置：MCP 能把上下文和工具送进来，但“哪些 server 开启、哪些 resource 可见、哪些 tool 暴露、何时列出、如何压缩工具输出、如何处理冲突”仍然是 harness / context engineering 的责任。

## 资料地图

| 资料 | 类型 | 主要贡献 |
|---|---|---|
| Anthropic: [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) | 官方工程博客 | 定义 context engineering；提出 context 有限、长程任务需要 compaction、structured note-taking、multi-agent architectures |
| LangChain: [Context Engineering](https://www.langchain.com/blog/context-engineering-for-agents) | 工程博客 | 把策略分成 write、select、compress、isolate |
| LangChain: [How agents can use filesystems for context engineering](https://www.langchain.com/blog/how-agents-can-use-filesystems-for-context-engineering) | 工程博客 | 把 filesystem 作为 agent 可读写的外部上下文 |
| Cognition: [Don't Build Multi-Agents](https://cognition.ai/blog/dont-build-multi-agents) | 工程博客 | 强调多 agent 的核心难点是 context sharing 和 implicit decisions |
| Manus: [Context Engineering for AI Agents: Lessons from Building Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus) | 工程案例 | KV-cache、稳定前缀、tool masking、filesystem memory、recitation、保留失败 |
| OpenAI Codex: [AGENTS.md](https://developers.openai.com/codex/guides/agents-md) / [Best practices](https://developers.openai.com/codex/learn/best-practices) | 官方文档 | 项目指令分层、减少无关上下文、测试和 review 反馈进入下一轮上下文 |
| MCP: [What is MCP?](https://modelcontextprotocol.io/docs/getting-started/intro) / [Architecture](https://modelcontextprotocol.io/docs/learn/architecture) | 官方文档 | 标准化 data、tools、prompts 的 context exchange |
| [Lost in the Middle](https://arxiv.org/abs/2307.03172) | 论文 | 长上下文不等于可靠使用长上下文 |
| [LongBench](https://arxiv.org/abs/2308.14508), [LongBench v2](https://arxiv.org/abs/2412.15204), [RULER](https://openreview.net/forum?id=kIoBbc76Sy) | benchmark | 评估长上下文理解、推理和真实可用窗口 |
| [SWE-agent](https://arxiv.org/abs/2405.15793), [OpenHands](https://arxiv.org/abs/2407.16741) | 开源项目 / 论文 | 表明 agent-computer interface、文件导航、命令执行和测试反馈会显著影响 coding agent 能力 |
| [Agentic Context Engineering](https://arxiv.org/abs/2510.04618) | 论文 | 把 context 当作可演化 playbook，通过生成、反思、整理来自我改进 |
| [Context Engineering for AI Agents in OSS](https://arxiv.org/abs/2510.21413) | 论文 | 研究开源项目中 agent 配置文件的采用与演化 |
| [OWASP MCP Top 10](https://owasp.org/www-project-mcp-top-10/) / [OWASP LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/) | 安全框架 | prompt injection、contextual payload、excessive agency、RAG 风险 |
| [Moffatt v. Air Canada, 2024 BCCRT 149](https://decisions.civilresolutionbc.ca/crt/crtd/en/item/525448/index.do) | 事故 / 法律案例 | chatbot 给出错误政策信息，企业被要求承担责任 |

## 核心框架：Write, Select, Compress, Isolate

LangChain 对 context engineering 的四分法很适合作为工程框架：

```text
Write    把信息写到上下文窗口之外
Select   在需要时选入相关上下文
Compress 把已有上下文压缩成更小表示
Isolate  用边界隔离不同上下文
```

### 1. Write：把上下文写到窗口之外

**事实**：Anthropic 提到 structured note-taking / agentic memory：agent 定期把 notes 写到 context window 外部，之后再拉回窗口。LangChain 的 filesystem 文章也把文件系统视为 agent 的上下文工程工具，deep agents 可以读、写、编辑、列出和搜索文件。

**工程做法**：

- 把长期事实写进 `docs/`，而不是长期留在聊天记录里。
- 把任务状态写成 `plans/*.md`、`status.md`、`todo.md`。
- 把临时搜索结果写入 `materials/`，只在 context 里保留路径和摘要。
- 把偏好、项目规则、review 标准写成可版本化文件。
- 把失败样本写成 eval case 或 regression test。

**推断**：write 的本质是把 context window 从“唯一记忆”降级为“当前工作集”。这和操作系统里 RAM / disk 的分层类似。

### 2. Select：按任务选择上下文

**事实**：OpenAI Codex 的 [AGENTS.md 文档](https://developers.openai.com/codex/guides/agents-md) 说明 Codex 会从全局、项目根目录到当前工作目录分层读取指令文件，靠近当前目录的指导出现在后面并覆盖更早指导；默认还有 `project_doc_max_bytes` 限制，文档建议拆分到嵌套目录来避免关键指导被截断。

**事实**：OpenAI Codex pricing FAQ 也建议控制 prompt 大小、减少不必要 context、缩小 `AGENTS.md`，并通过嵌套 `AGENTS.md` 控制大项目注入多少上下文。

**工程做法**：

- 根 `AGENTS.md` 只放入口地图和全局规则。
- 子目录放局部规则，例如 `frontend/AGENTS.md`、`services/payments/AGENTS.md`。
- 大文档只提供目录和摘要，按需读取章节。
- RAG 检索结果要 rerank、去重、保留来源，而不是盲目塞 top-k。
- 工具列表要最小化，避免功能重叠导致模型选择困难。

**推断**：select 的目标不是“让模型知道尽可能多”，而是“让模型在当前一步无需猜测”。这意味着上下文选择应围绕当前 action，而不是围绕整个项目百科。

### 3. Compress：压缩上下文

**事实**：Anthropic 将 compaction 定义为：当 conversation 接近 context window 限制时，总结内容并用 summary 重新启动一个新的 context window。其 Claude Code 示例会保留架构决策、未解决 bug 和实现细节，同时丢弃冗余工具输出或消息。

**事实**：OpenAI Codex 也提供 `/compact`、conversation compaction、context window 管理等能力。本项目已有单独文档解释 Codex compaction，关键点是：压缩不是无损 zip，也不是可靠事实来源；长期事实仍应落到代码、测试、文档和计划文件。

**工程做法**：

- 对工具输出做“结果清理”：保留错误类型、关键路径、失败断言，丢弃重复日志。
- 对长会话做阶段性摘要：目标、已完成、未完成、关键约束、文件路径、验证结果。
- 对检索文档做分层摘要：一行摘要、段落摘要、原文链接。
- 对多 agent 结果做结构化汇总：结论、证据、风险、未确认项、下一步。

**风险**：

- 压缩可能丢掉后来才变重要的细节。
- summary 会引入解释偏差。
- 错误事实进入 summary 后会污染后续推理。
- 如果没有 trace，后续很难审计 summary 为什么这么写。

### 4. Isolate：隔离上下文

**事实**：LangChain 将 isolate 视为 context engineering 的一类策略。Cognition 则从反面指出，简单多 agent 方案很脆弱：如果只把任务拆给多个 subagent，subagent 会缺少完整 trace；如果并行工作时看不到彼此行动，又会产生冲突的隐式决策。

**工程做法**：

- 用 subagent 隔离高噪声探索，例如长日志分析、代码库调研、benchmark 跑分。
- 用只读 reviewer agent 隔离 review 上下文，避免被实现过程影响。
- 用不同 worktree / sandbox 隔离并行实现，降低文件冲突。
- 用 `agent-as-tool` 让主 agent 保持任务所有权，只接收结构化结果。

**推断**：isolate 的价值不是“多几个 AI 更聪明”，而是减少上下文污染、控制权限和降低主上下文负担。多 agent 的代价是 context handoff 变难，因此必须用结构化交接和 trace 支撑。

## 工程案例

### Anthropic：把 context 当有限注意力预算

**事实**：Anthropic 的核心表述是：好的 context engineering 意味着找到最小的高信号 token 集合，以最大化目标结果概率。它讨论了 system prompt、tools、examples、knowledge、actions 和 observations 等上下文组成。

**事实**：对长程任务，Anthropic 提出三类技术：

- compaction。
- structured note-taking / agentic memory。
- sub-agent architectures。

**观点**：Anthropic 认为，即使模型继续变强，维护长交互中的 coherence 仍会是构建有效 agent 的中心问题。

**推断**：这和传统软件的性能优化类似：更快 CPU 不意味着可以停止做缓存、索引和内存布局；更强模型也不意味着可以停止做上下文布局。

### LangChain：把上下文治理拆成四个操作

**事实**：LangChain 将 agent context engineering 分为 write、select、compress、isolate，并结合 agent 产品和论文解释每类策略。

**事实**：LangChain 的 filesystem 文章明确指出，agent 失败可能因为模型不够好，也可能因为没有正确上下文；filesystem 工具让 agent 能在外部读写、编辑和搜索上下文。

**观点**：LangChain 的框架价值在于把“上下文”从抽象词拆成可实现操作。

**推断**：对个人 coding agent 工作流，最容易立刻落地的是：

- write：把计划和状态写文件。
- select：根文档只做入口地图。
- compress：长日志只保留关键失败。
- isolate：把调研和长输出交给 subagent 或独立会话。

### Manus：把 context 当生产成本和延迟问题

**事实**：Manus 官方文章在 2025-07-18 分享了几个经验：围绕 KV-cache 设计、保持 prompt prefix 稳定、上下文 append-only、不要动态移除工具而是 mask、把文件系统作为外部上下文、通过 recitation 维持目标注意力、保留失败动作、避免 few-shot 造成行为单一化。

**观点**：Manus 的重要观点是：生产 agent 的关键指标不只是 task success，还包括 KV-cache hit rate，因为它直接影响延迟和成本。它把上下文结构设计成经济性问题，而不只是质量问题。

**推断**：Manus 的经验尤其适合长程 agent：

- 稳定前缀类似 API ABI，频繁变动会破坏缓存。
- append-only event stream 比每轮重排上下文更可缓存、可审计。
- tool definitions 不应按状态频繁变动；状态约束更适合放在权限层、mask 层或 runtime policy。
- 文件系统不仅是工具，也是 agent 的外部工作记忆。

**注意**：Manus 原文是工程案例，不是可复现实验论文；商业数字和成本降幅不应直接外推成通用结论。

### Cognition：多 agent 的难点是上下文一致性

**事实**：Cognition 的文章提出两个原则：

- share context, and share full agent traces, not just individual messages。
- actions carry implicit decisions, and conflicting decisions carry bad results。

**观点**：Cognition 因此建议默认避免不满足这些原则的多 agent 架构。它更偏好连续上下文的单线程 agent，除非长任务确实需要更复杂的上下文结构。

**推断**：这不是反对 subagent，而是提醒：subagent 是 context engineering 工具，不是自动提升质量的组织结构。并行化带来的速度收益，必须和 context handoff、隐式决策冲突、review 成本一起评估。

### OpenAI Codex：项目指令、测试和 review 都是上下文

**事实**：OpenAI Codex 的 AGENTS.md 文档说明，Codex 启动时会构建 instruction chain：全局 scope、项目 scope、从根目录到当前工作目录逐层合并，并受默认 32 KiB 的 `project_doc_max_bytes` 限制。

**事实**：Codex best practices 建议不要只要求 Codex 改代码，还要让它创建测试、运行相关检查、确认结果、review diff；diff panel 的行级反馈会作为下一轮 Codex 的上下文。

**推断**：Codex 的实践说明 coding agent 的 context 不应只由用户手动喂给模型。仓库文件、测试命令、review 标准、diff 反馈和工具输出都应该成为结构化上下文来源。

## 论文、Benchmark 和 Eval

### 长上下文能力评估

**事实**：

- 《[Lost in the Middle](https://arxiv.org/abs/2307.03172)》发现模型对位于上下文开头或结尾的信息使用更好，对中间信息更容易失败。
- 《[LongBench](https://arxiv.org/abs/2308.14508)》提出双语、多任务长上下文理解 benchmark，并指出当时模型仍然在更长上下文上挣扎。
- 《[LongBench v2](https://arxiv.org/abs/2412.15204)》进一步关注真实长上下文多任务中的深度理解和推理。
- [RULER](https://openreview.net/forum?id=kIoBbc76Sy) 关注“真实可用 context size”，不是只看模型标称窗口。

**推断**：这些 benchmark 对 context engineering 的启发是：不要用厂商标称 context window 当工程预算。应该测：

- 关键事实放在不同位置时模型是否还能用。
- context 增长时正确率如何下降。
- 检索片段数量增加时是否引入噪声。
- 压缩摘要是否保留任务关键约束。
- 同一个任务在不同 context policy 下成本、延迟、正确率如何变化。

### Coding agent 评估

**事实**：[SWE-agent](https://arxiv.org/abs/2405.15793) 的论文标题就强调 Agent-Computer Interface，认为为 LM 设计的接口能显著增强 agent 创建和编辑代码、导航仓库、运行测试和程序的能力。[OpenHands](https://arxiv.org/abs/2407.16741) 则提供一个让 AI software developer 像人类开发者一样写代码、操作命令行、浏览网页的平台。

**事实**：OpenAI 的 [SWE-bench Verified](https://openai.com/index/introducing-swe-bench-verified/) 说明 SWE-bench 是评估 LLM 解决 GitHub 真实 issue 能力的流行套件，而 Verified 版本通过人工软件工程师审查子集来提升可靠性。

**推断**：SWE-bench 这类分数不是纯模型能力分数，而是：

```text
model + scaffold / harness + context selection + tools + retries + tests + patch strategy
```

这和本项目已有 harness engineering 调研一致：context engineering 应该被纳入 agent eval 配置，而不是只在 prompt 里手调。

### Context 自我演化

**事实**：《[Agentic Context Engineering](https://arxiv.org/abs/2510.04618)》把 context 视为 evolving playbook，通过 generation、reflection、curation 来积累、提炼和组织策略。

**事实**：《[Context Engineering for AI Agents in Open-Source Software](https://arxiv.org/abs/2510.21413)》研究了 466 个开源项目中 AI configuration files 的采用、内容、呈现方式和演化。

**推断**：研究方向正在从“人工写 context”走向“agent 通过 trace 和反馈维护 context”。但在生产系统里，这必须受版本化、review、回滚、eval 和权限控制约束，否则会变成 memory poisoning 或 prompt drift。

## 开源项目和工具生态

### SWE-agent

**事实**：SWE-agent 明确把 Agent-Computer Interface 作为研究对象：模型固定时，接口设计会影响 agent 在软件工程任务上的表现。

**对 context engineering 的启发**：

- 文件导航命令本身就是 context selection interface。
- 编辑命令决定模型如何引用和修改代码。
- 测试命令输出是 high-value context，但需要裁剪。
- ACI 比“直接给 shell”更可控。

### OpenHands

**事实**：OpenHands / OpenDevin 是开源 AI software developer 平台，让 agent 通过写代码、命令行和浏览器与世界交互。

**对 context engineering 的启发**：

- Agent workspace、sandbox、browser、terminal 都会产生上下文。
- 平台化 agent 需要统一管理 history、state、tool output、human feedback。
- 开源 agent 的可复现性取决于能否记录和重放这些上下文。

### Letta / MemGPT

**事实**：[MemGPT](https://arxiv.org/abs/2310.08560) 提出让 LLM 管理不同层级记忆，以突破有限 context window。Letta 是其后续开源 agent memory / state 管理系统方向之一。

**对 context engineering 的启发**：

- memory 不是一个无限聊天记录。
- memory 需要分层：工作记忆、长期事实、档案、工具结果。
- 何时写入、何时检索、何时遗忘，比“有没有记忆”更关键。

### MCP

**事实**：MCP 正在成为连接 tools、resources、prompts 的协议层。

**推断**：MCP server 越多，context engineering 越重要。因为每个 server 都可能带来：

- 新工具描述。
- 新 resource。
- 新权限边界。
- 新 prompt / template。
- 新攻击面。

因此“装更多 MCP”不等于 agent 更聪明，可能只是让 context window 更拥挤、工具选择更模糊、权限更难审计。

## 开发者社区讨论

### 共同趋势

**事实**：Hacker News、Reddit、X、独立博客中，“context engineering is the new prompt engineering”已经成为高频说法。社区讨论常聚焦：

- memory 是否会成为 agent 标配。
- prompt engineer 是否会被 context engineer / AI engineer 替代。
- 多 agent 是否值得。
- RAG 是否被过度包装。
- 长上下文模型是否让检索不再必要。

**观点**：社区里有两种相反声音：

- 支持者认为 context engineering 更准确描述了生产 AI 工程工作。
- 怀疑者认为它只是把 retrieval、memory、prompt、workflow 重新包装成新名词。

**推断**：两边都有道理。作为术语，context engineering 确实有 hype 成分；作为工程问题，它不是新问题，但 agent 时代让它变得更集中、更关键、更需要系统化。

### 社区里的有效提醒

**观点**：多个社区讨论都提醒：大多数团队没有量化 context strategy 的效果，只凭“感觉更好”改 prompt、加 memory、加 vector store。

**推断**：这正是 context engineering 和普通 prompt hacking 的分界：

```text
prompt hacking: 改了试试，看起来不错
context engineering: 定义上下文策略 -> 记录 trace -> 跑 eval
                    -> 比较质量/成本/延迟/安全性 -> 版本化
```

## 招聘市场信号

**事实**：截至 2026-05-30，公开招聘中已经能看到 `Context Engineer` 或 `Prompt Engineer / Context Engineer` 这样的标题。例如 Contextual AI 的 careers 页面列出 `Context Engineer`；一些招聘聚合页面也出现 `Software AI Engineer, Context Engineering`、`LLM Context Engineer`、`Prompt Engineer, Agent Prompts & Evals` 等表述。

**事实**：同时，职位数量仍远少于 `AI Engineer`、`ML Engineer`、`Applied AI Engineer`、`Prompt Engineer`、`Solutions Engineer`、`Agent Engineer`。

**推断**：招聘市场的信号是：

- `Context Engineer` 正在出现，但还不是稳定主流职称。
- 很多岗位把 context engineering 藏在 AI engineer / prompt engineer / solutions engineer / agent engineer 的职责里。
- 真正稀缺的不是“会写提示词”，而是能把业务知识、检索、工具、权限、eval、observability 和产品流程接起来的人。

**推断**：如果用能力模型描述，context engineer 更像：

```text
AI application engineer
+ information architect
+ eval engineer
+ developer tooling engineer
+ product / domain analyst
+ security-aware systems engineer
```

而不是只会写自然语言提示词的人。

## 事故复盘和安全风险

### Air Canada chatbot：错误上下文也会变成业务承诺

**事实**：在 [Moffatt v. Air Canada, 2024 BCCRT 149](https://decisions.civilresolutionbc.ca/crt/crtd/en/item/525448/index.do) 中，Air Canada 网站 chatbot 给出关于 bereavement fare retroactive application 的错误信息。British Columbia Civil Resolution Tribunal 在 2024-02-14 的决定中认定 Air Canada 对 chatbot 信息承担责任，并判给申请人部分退款、利息和费用。

**推断**：这个案例不是典型 agent harness 事故，但对 context engineering 很有价值：

- 客服 AI 的上下文必须来自可审计、最新、权威的政策源。
- 如果 AI 输出和官方政策页面冲突，系统要能检测、拒答或升级人工。
- “聊天机器人只是建议”在用户体验和法律责任上未必成立。

### Prompt injection 和 context poisoning

**事实**：OWASP LLM Top 10 将 prompt injection、sensitive information disclosure、excessive agency 等列为 LLM 应用风险。OWASP MCP Top 10 也列出 contextual payload、tool poisoning、context injection、over-sharing 等 MCP 相关风险。

**观点**：UK NCSC 曾警告，LLM 当前并不真正区分 prompt 中的 instruction 和 data；prompt injection 可能是 LLM 技术的内在问题之一，系统设计应降低影响，而不是幻想单一修复。

**推断**：context engineering 的安全原则：

- 外部内容默认不可信。
- RAG 文档、网页、issue、邮件、PDF 都可能携带恶意指令。
- memory 写入必须有过滤、来源、审批和回滚。
- tool output 不应无条件进入下一轮高优先级上下文。
- 高风险工具必须靠 runtime 权限限制，而不是靠 prompt 说“不要”。

## 历史类比

### 1. 操作系统：context window 像 RAM

**推断**：LLM 像 CPU，context window 像 RAM，文件系统、数据库、vector store、日志和文档像 disk / storage。Context engineering 对应：

- 内存布局。
- cache policy。
- paging。
- garbage collection。
- process isolation。
- 权限控制。

这个类比很有用，但不能过度使用：LLM 的“内存访问”不是确定性的，注意力不是指针寻址。

### 2. 编译器：prompt 只是源代码的一部分

**推断**：一次 agent 调用像一次编译 / 执行：

```text
source prompt
+ imports / docs
+ macros / templates
+ linked tools
+ runtime state
+ compiler flags / model params
-> model output / tool calls
```

所以只看 prompt 文本，就像只看一个源文件而忽略 include path、linker flags 和 runtime environment。

### 3. 数据工程：上下文质量决定输出质量

**推断**：RAG 和 memory 层的问题很像数据管道：

- schema 不清会导致工具误用。
- 数据过期会导致错误答案。
- provenance 缺失会导致无法审计。
- 去重和排序差会导致噪声。
- 数据污染会导致后续输出污染。

因此 context engineering 需要 data engineering 的 discipline：版本、血缘、质量检查、回放、监控。

### 4. Test harness：eval 是上下文策略的回归测试

**推断**：每次改 context policy 都可能改变 agent 行为。它需要像代码一样测试：

- 检索 top-k 改了，会不会引入错误来源？
- 压缩 prompt 改了，会不会丢关键约束？
- 新 MCP server 加入后，会不会让工具选择变差？
- 新 memory 规则会不会把临时偏好变成长期事实？

Context regression test 应该成为 agent harness 的一部分。

## 实践清单

### 项目文档

- 根 `AGENTS.md` 只放入口地图、全局规则和协作约定。
- 主题细节放到 `docs/<topic>/`。
- 子目录规则就近放置，避免全局上下文膨胀。
- 文档要写“如何验证”，不要只写“应该怎么做”。
- 对会变化的信息标日期和来源。

### Agent runtime

- 记录每轮实际发送给模型的 context。
- 记录检索来源、排序分、截断策略。
- 记录 tool schema 和 tool output 的 token 成本。
- 对大工具输出做默认裁剪和可追溯落盘。
- 把 context assembly 做成可测试模块，而不是散落在 prompt 字符串拼接里。

### Memory

- 区分短期任务状态、长期用户偏好、项目事实、失败经验。
- memory 写入要有来源、时间、作用域和可信度。
- 不要让模型无审批地改全局长期记忆。
- 支持删除、回滚、过期和冲突解决。

### Retrieval

- 不只做向量相似度，必要时结合 keyword、symbol search、AST、git history、docs map。
- 返回片段必须带路径、标题、日期、来源。
- 评估 top-k、chunk size、rerank、去重和引用位置。
- 对“不足以回答”的场景训练拒答或继续检索。

### Compression

- 压缩必须保留：目标、约束、关键事实、文件路径、决策、未完成事项、验证状态。
- 对代码任务保留具体 symbol、路径、命令和失败断言。
- 对调研任务保留来源链接和时间。
- 压缩摘要本身要可审计，不能成为唯一事实来源。

### Tool 和 MCP

- 工具名称、描述、参数 schema 要清晰且低重叠。
- 不要把所有 MCP server 默认打开。
- 对高风险工具做权限分层和 human approval。
- 对 tool output 做信任分级：内部命令输出、外部网页、用户输入、模型生成内容不能同等对待。

### Eval

- 为 context policy 建 eval，而不是只为模型建 eval。
- 同一模型下比较不同 context 策略的质量、成本、延迟。
- 加入 adversarial context：冲突来源、过期文档、恶意 prompt injection、超长噪声。
- 对长程任务测 compaction 后是否还能继续。

## 对本资料库的落地建议

**事实**：本仓库已经把 `AGENTS.md` 设计成入口地图，并把主题内容拆到 `docs/`。这符合 OpenAI Codex AGENTS.md 分层和 Anthropic / LangChain 对上下文有限性的建议。

**建议**：

- 新增正式文档时继续同步根 README 和主题 README。
- 参考资料继续集中到 `docs/harness-engineering/references.md`。
- 长调研材料未来可放到 `materials/context-engineering/`，正式结论再进入 `docs/`。
- 对易变信息使用“调研日期 + 来源类型 + 可信度”。
- 建一个 context engineering 评估清单，用来审查 `AGENTS.md`、主题 README、plans 和 references 是否会给 agent 提供过多或过少上下文。

## 最短判断框架

当 agent 失败时，不要只问：

```text
这个 prompt 怎么改？
```

应该问：

```text
这是哪类 context failure？

1. 缺上下文：模型没有看到必要事实。
2. 噪声过多：模型看到了太多无关内容。
3. 上下文过期：模型看到旧事实。
4. 上下文冲突：多个来源互相矛盾。
5. 上下文无来源：模型无法判断可信度。
6. 上下文污染：错误、幻觉或攻击进入了记忆。
7. 上下文丢失：压缩、截断或新会话丢了关键状态。
8. 上下文越权：工具、secret、生产数据进入了不该进入的窗口。
9. 上下文不可验证：没有 trace / eval 证明这个策略有效。
```

对应修复方式不是只改 wording，而是改：

- 文档结构。
- 检索策略。
- 工具输出格式。
- memory 写入规则。
- compaction prompt。
- MCP server 配置。
- 权限策略。
- eval case。
- trace 和 observability。

## 结论

**事实**：从 Anthropic、LangChain、OpenAI Codex、MCP、SWE-agent、OpenHands、Manus、Cognition 和长上下文 benchmark 看，context engineering 已经是 agent 工程里的真实问题集合。

**观点**：它是否会成为长期稳定职称还不确定，但它作为工程能力会继续存在。

**推断**：最务实的理解是：

```text
Prompt engineering 优化一句话。
Context engineering 优化模型每一步的工作记忆。
Harness engineering 优化 agent 完成任务的整个运行系统。
```

对 coding agent 来说，未来真正的竞争点不是“谁能写最长的 prompt”，而是谁能把仓库知识、工具、状态、测试、trace、权限和 review 组织成模型可用、可验证、可维护的上下文系统。
