# AI Agent Notes

这个项目用于整理 **harness engineering、coding agent、AI agent、模型代码能力、agent eval** 等相关资料、调研文档和实践流程。

现在文档按主题拆分：

1. [`docs/harness-engineering/`](docs/harness-engineering/README.md)：agent harness 的概念、分层方法、实践案例和参考资料。
2. [`docs/coding-agents/`](docs/coding-agents/README.md)：code CLI、coding agent 插件、个人 TDD 工作流、Codex 上下文压缩、Memories、窗口管理、sandbox、高手使用经验。
3. [`docs/ai-models/`](docs/ai-models/README.md)：模型代码能力训练、reasoning、benchmark、模型报告解读和推理基础设施。
4. [`docs/llm-api/`](docs/llm-api/README.md)：LLM API 形态、工具调用、结构化输出、eval、开源生态、工程案例、事故复盘和招聘市场。

推荐阅读路径：

1. 先读 [`docs/harness-engineering/foundations/01-what-is-harness-engineering.md`](docs/harness-engineering/foundations/01-what-is-harness-engineering.md)，建立 harness engineering 的基本概念。
2. 再读 [`docs/coding-agents/research/workflows/01-expert-coding-agent-workflows.md`](docs/coding-agents/research/workflows/01-expert-coding-agent-workflows.md)，理解高手如何实际使用 coding agent。
3. 如果要把方法落地到新项目，读 [`docs/coding-agents/agent/playbooks/workflows/new-project-bootstrap.md`](docs/coding-agents/agent/playbooks/workflows/new-project-bootstrap.md)。
4. 如果要接入既有项目，读 [`docs/coding-agents/agent/playbooks/workflows/existing-project-onboarding.md`](docs/coding-agents/agent/playbooks/workflows/existing-project-onboarding.md)。
5. 如果要理解什么时候该用 subagent，读 [`docs/coding-agents/research/orchestration/01-when-to-use-subagents.md`](docs/coding-agents/research/orchestration/01-when-to-use-subagents.md)。
6. 如果要让 AI 生成代码更容易可靠 review，读 [`docs/coding-agents/research/workflows/02-ai-assisted-code-review.md`](docs/coding-agents/research/workflows/02-ai-assisted-code-review.md)。
7. 如果要设计 roadmap、project-status 和 plans，读 [`docs/coding-agents/research/workflows/03-plan-documents-for-coding-agents.md`](docs/coding-agents/research/workflows/03-plan-documents-for-coding-agents.md)。
8. 如果要理解 Claude Code / Codex CLI hooks 如何把规则变成 runtime 约束，读 [`docs/coding-agents/research/runtime/03-coding-agent-hooks.md`](docs/coding-agents/research/runtime/03-coding-agent-hooks.md)。
9. 如果要理解人人皆可 coding 后真正壁垒在哪里，读 [`docs/coding-agents/research/strategy/01-real-barriers-when-everyone-can-code.md`](docs/coding-agents/research/strategy/01-real-barriers-when-everyone-can-code.md)。
10. 如果要理解 Prompt Engineering 如何从提示词技巧演化到 context、eval 和 harness，读 [`docs/harness-engineering/layers/01-prompt-engineering-research.md`](docs/harness-engineering/layers/01-prompt-engineering-research.md)。
11. 如果要理解 Tool Use 如何从函数调用演化为 agent action layer，读 [`docs/harness-engineering/layers/02-tool-use-research.md`](docs/harness-engineering/layers/02-tool-use-research.md)。
12. 如果要理解 Agent Loop 如何从 ReAct 循环演化到生产级 agent runtime，读 [`docs/harness-engineering/layers/03-agent-loop-research.md`](docs/harness-engineering/layers/03-agent-loop-research.md)。
13. 如果要理解 Context Engineering 如何治理 agent 每一步能看到什么，读 [`docs/harness-engineering/layers/04-context-engineering-research.md`](docs/harness-engineering/layers/04-context-engineering-research.md)。
14. 如果要理解 Agent Planning 如何从 prompt 技巧变成 harness 控制面，读 [`docs/harness-engineering/layers/05-planning-research.md`](docs/harness-engineering/layers/05-planning-research.md)。
15. 如果要理解 Agent Skills 如何把重复 prompt、组织知识、工作流和验证封装成可复用能力包，读 [`docs/coding-agents/research/runtime/04-agent-skills.md`](docs/coding-agents/research/runtime/04-agent-skills.md)。
16. 如果要系统研究 Multi-Agent，读 [`docs/coding-agents/research/orchestration/02-multi-agent-research.md`](docs/coding-agents/research/orchestration/02-multi-agent-research.md)。
17. 如果要理解 Vibe Coding 中非专家使用 AI 写陌生技术栈的 review 困境和学习债，读 [`docs/coding-agents/research/workflows/04-vibe-coding-review-and-learning-debt.md`](docs/coding-agents/research/workflows/04-vibe-coding-review-and-learning-debt.md)。
18. 如果要理解 Anthropic / Claude Code Dynamic workflows 如何把大量 subagents 编排成可运行脚本，读 [`docs/coding-agents/research/workflows/06-anthropic-dynamic-workflows.md`](docs/coding-agents/research/workflows/06-anthropic-dynamic-workflows.md)。
19. 如果要把 Vibe Coding 的判断落实成可复制 prompt，读 [`docs/coding-agents/agent/playbooks/workflows/learning-first-vibe-coding.md`](docs/coding-agents/agent/playbooks/workflows/learning-first-vibe-coding.md)。
20. 如果要用费曼学习法学习编程和这个仓库的资料，读 [`docs/coding-agents/agent/playbooks/workflows/feynman-learning-for-programming.md`](docs/coding-agents/agent/playbooks/workflows/feynman-learning-for-programming.md)。
21. 如果要研究 Codex CLI 实验 Memories 功能，读 [`docs/coding-agents/codex/memory/01-codex-cli-experimental-memories.md`](docs/coding-agents/codex/memory/01-codex-cli-experimental-memories.md)。
22. 如果要理解 LLM API 如何进入生产系统，读 [`docs/llm-api/research/01-llm-api-research.md`](docs/llm-api/research/01-llm-api-research.md)。
23. 如果要理解 reasoning、模型能力和 benchmark，再读 [`docs/ai-models/README.md`](docs/ai-models/README.md) 下的文档。
24. 如果要理解长上下文、prompt caching 和 agent runtime 成本，再读 [`docs/ai-models/infrastructure/01-kv-cache-research.md`](docs/ai-models/infrastructure/01-kv-cache-research.md)。

一句话概括：

> 这个项目关注的不只是“模型会不会写代码”，而是模型、工具、上下文、测试、权限、review 和工作流怎样组成一个可持续的软件工程系统。
