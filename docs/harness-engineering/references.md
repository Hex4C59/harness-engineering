# 参考资料

本文档目录主要参考以下资料，并结合 noclaw 项目目标做了重新组织。

## 核心资料

- OpenAI Engineering Blog: [Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/)
  - 关键点：人类负责 steering，agent 负责 execution；仓库知识作为事实来源；让 UI、日志、指标对 agent 可读；用 lint、结构测试和文档维护防止漂移。
- OpenAI Engineering Blog: [Unlocking the Codex harness: how we built the App Server](https://openai.com/index/unlocking-the-codex-harness/)
  - 关键点：Codex harness 包含 agent loop、thread lifecycle、持久化、配置、认证、sandbox 工具执行和扩展；App Server 把 harness 暴露为客户端友好的协议。
- LangChain Blog: [The Anatomy of an Agent Harness](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness)
  - 关键点：`Agent = Model + Harness`；harness 是模型之外的代码、配置和执行逻辑；文件系统、规划、工具输出管理、skills、verification loop 都属于 harness 设计。
- arXiv: [AI Harness Engineering: A Runtime Substrate for Foundation-Model Software Agents](https://arxiv.org/abs/2605.13357)
  - 关键点：把软件工程能力视为 model-harness-environment 系统的结果；提出 task specification、context selection、tool access、project memory、task state、observability、failure attribution、verification、permissions、entropy auditing、intervention recording 等责任。

## Prompt Engineering 相关资料

- OpenAI Docs: [Prompt engineering](https://developers.openai.com/api/docs/guides/prompt-engineering)
  - 关键点：prompt 是使模型稳定满足需求的有效指令；复杂应用应固定模型快照并建立 eval。
- OpenAI Docs: [Evaluation best practices](https://developers.openai.com/api/docs/guides/evaluation-best-practices)
  - 关键点：eval 是处理生成式 AI 非确定性的基础工程机制，应围绕真实任务、日志样本和持续评估构建。
- Anthropic Docs: [Prompt engineering overview](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview)
  - 关键点：prompt engineering 是提升 Claude 输出质量的方法之一，但 latency、cost 等失败模式不应靠 prompt 修。
- Google AI for Developers: [Prompt design strategies](https://ai.google.dev/guide/prompt_best_practices)
  - 关键点：直接、结构化、带约束的 prompt 更适合 Gemini；近期事实和计算应交给 grounding / code execution。
- OWASP: [Top 10 for Large Language Model Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
  - 关键点：prompt injection 是 LLM 应用的核心风险，不能只靠 prompt 防御。
- arXiv: [The Prompt Report](https://arxiv.org/abs/2406.06608)
  - 关键点：系统整理 prompt 技术、词汇和应用场景，可作为 prompt engineering 术语地图。
- arXiv: [Promptware Engineering: Software Engineering for Prompt-Enabled Systems](https://arxiv.org/abs/2503.02400)
  - 关键点：把 prompt-enabled software 视作需要需求、测试、维护和安全治理的软件工程对象。

## Context Engineering 相关资料

- Anthropic Engineering Blog: [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
  - 关键点：context engineering 是在 LLM 推理期间维护最佳 token 集合的策略；context 是有限注意力预算；长程任务需要 compaction、structured note-taking 和 sub-agent architectures。
- LangChain Blog: [Context Engineering](https://www.langchain.com/blog/context-engineering-for-agents)
  - 关键点：把 agent context engineering 策略分成 write、select、compress、isolate。
- LangChain Blog: [How agents can use filesystems for context engineering](https://www.langchain.com/blog/how-agents-can-use-filesystems-for-context-engineering)
  - 关键点：文件系统可以作为 agent 外部上下文和工作记忆，支持读、写、编辑、列出和搜索。
- Cognition Blog: [Don't Build Multi-Agents](https://cognition.ai/blog/dont-build-multi-agents)
  - 关键点：多 agent 的脆弱点在 context sharing 和 implicit decisions；应分享完整 agent traces，而不是只分享单条消息。
- Manus Blog: [Context Engineering for AI Agents: Lessons from Building Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)
  - 关键点：KV-cache hit rate、稳定 prompt prefix、append-only context、mask 而不是移除工具、filesystem memory、recitation、保留失败记录。
- OpenAI Codex Docs: [Custom instructions with AGENTS.md](https://developers.openai.com/codex/guides/agents-md)
  - 关键点：Codex 分层读取全局和项目 `AGENTS.md`；大项目应拆分指令，避免上下文截断。
- OpenAI Codex Docs: [Best practices](https://developers.openai.com/codex/learn/best-practices)
  - 关键点：让 Codex 创建测试、运行检查、review diff；review 反馈会进入后续上下文。
- Model Context Protocol: [What is MCP?](https://modelcontextprotocol.io/docs/getting-started/intro) 和 [Architecture overview](https://modelcontextprotocol.io/docs/learn/architecture)
  - 关键点：MCP 标准化 AI 应用和外部 data sources、tools、prompts 的连接；MCP 本身不规定 AI 应用如何管理上下文。
- arXiv: [Lost in the Middle: How Language Models Use Long Contexts](https://arxiv.org/abs/2307.03172)
  - 关键点：长上下文模型不一定能均匀使用上下文，相关信息位于中间时可能显著退化。
- arXiv: [LongBench](https://arxiv.org/abs/2308.14508), [LongBench v2](https://arxiv.org/abs/2412.15204), OpenReview: [RULER](https://openreview.net/forum?id=kIoBbc76Sy)
  - 关键点：评估长上下文理解、推理和真实可用窗口，提醒不要把标称窗口当成真实能力。
- arXiv: [SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering](https://arxiv.org/abs/2405.15793)
  - 关键点：Agent-Computer Interface 设计会显著影响 coding agent 的仓库导航、文件编辑和测试执行能力。
- arXiv: [OpenHands: An Open Platform for AI Software Developers as Generalist Agents](https://arxiv.org/abs/2407.16741)
  - 关键点：software agent 平台需要让 agent 像开发者一样写代码、使用命令行和浏览网页。
- arXiv: [Agentic Context Engineering](https://arxiv.org/abs/2510.04618)
  - 关键点：把 context 视为可演化 playbook，通过 generation、reflection、curation 自我改进。
- arXiv: [Context Engineering for AI Agents in Open-Source Software](https://arxiv.org/abs/2510.21413)
  - 关键点：研究 466 个开源项目中 AI configuration files 的采用、内容和演化。
- OWASP: [Top 10 for Large Language Model Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/) 和 [MCP Top 10](https://owasp.org/www-project-mcp-top-10/)
  - 关键点：prompt injection、RAG / embedding 风险、contextual payload、excessive agency 等都是 context engineering 必须处理的安全风险。
- British Columbia Civil Resolution Tribunal: [Moffatt v. Air Canada, 2024 BCCRT 149](https://decisions.civilresolutionbc.ca/crt/crtd/en/item/525448/index.do)
  - 关键点：客服 chatbot 给出错误政策信息后，企业仍可能承担责任；上下文来源、时效和权威性是业务风险。

## Planning 相关资料

- arXiv: [PlanBench: An Extensible Benchmark for Evaluating Large Language Models on Planning and Reasoning about Change](https://arxiv.org/abs/2206.10498)
  - 关键点：用经典 planning domain 系统评估 LLM 的动作、状态变化和计划生成能力，避免把常识检索误当规划能力。
- arXiv: [TravelPlanner: A Benchmark for Real-World Planning with Language Agents](https://arxiv.org/abs/2402.01622)
  - 关键点：旅行规划暴露多约束、工具选择、信息收集和长期一致性问题。
- Anthropic Docs: [Claude Code permission modes](https://code.claude.com/docs/en/permission-modes)
  - 关键点：Plan Mode 是 read-only 权限模式，用于修改前探索代码和提出计划。
- OpenAI Docs: [Trace grading](https://platform.openai.com/docs/guides/trace-grading)
  - 关键点：对 agent trace 中的决策、工具调用和 reasoning steps 打分，用于定位 planning 和执行失败。
- Microsoft AutoGen Docs: [Magentic-One](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/magentic-one.html)
  - 关键点：Orchestrator 使用 plan、Task Ledger 和 Progress Ledger 跟踪多 agent 任务并动态重规划。
- Google ADK Docs: [Loop agents](https://google.github.io/adk-docs/agents/workflow-agents/loop-agents/)
  - 关键点：把 deterministic workflow agents 和 LLM-driven dynamic routing 分开，适合表达可控迭代流程。

## 相关但需谨慎阅读

- 社区文章和博客对 harness engineering 的定义仍在快速变化，适合做启发，不宜当作唯一权威。
- “Harness” 也可能指 Harness.io 这个软件交付平台，或传统软件测试里的 test harness；本目录不讨论这些含义。

## 对 noclaw 最有价值的抽象

- `AGENTS.md` 应是入口地图，不是百科全书。
- `docs/` 应成为 agent 可读的事实来源。
- 实施计划、当前状态、决策记录要版本化。
- 工具、权限、状态、验证和反馈闭环比单次 prompt 更重要。
- 当 agent 失败时，优先问“缺少什么 harness 能力”，而不只是“怎么改提示词”。
