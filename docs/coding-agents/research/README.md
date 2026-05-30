# Coding Agent Research

这里放跨工具、跨产品的 coding agent 调研、方法论、机制分析和案例研究。

这些文档主要用于阅读、理解和提炼规则。需要落地时，把其中稳定规则提炼到项目的 `AGENTS.md`、`docs/development.md`、`docs/testing.md` 或 playbook。

## 主题地图

1. [`evaluation/`](evaluation/README.md)：如何比较 Code CLI、模型组合和 harness 配置。
2. [`runtime/`](runtime/README.md)：memory、sandbox、hooks、skills 等 agent runtime 机制。
3. [`orchestration/`](orchestration/README.md)：subagent、multi-agent 和编排边界。
4. [`workflows/`](workflows/README.md)：个人和团队工作流、review、计划文档、学习优先实践。
5. [`case-studies/`](case-studies/README.md)：具体工具、插件和 skill library 的设计分析。
6. [`strategy/`](strategy/README.md)：coding agent 时代的软件工程壁垒和长期判断。

## 推荐阅读

1. [`workflows/01-expert-coding-agent-workflows.md`](workflows/01-expert-coding-agent-workflows.md)：编程高手如何使用 Coding Agent 的经验调研。
2. [`evaluation/01-how-to-evaluate-code-cli-and-models.md`](evaluation/01-how-to-evaluate-code-cli-and-models.md)：如何比较 code CLI 和模型组合。
3. [`runtime/01-coding-agent-memory-mechanisms.md`](runtime/01-coding-agent-memory-mechanisms.md)：Coding Agent 记忆机制综述。
4. [`runtime/02-coding-agent-sandbox-mechanisms.md`](runtime/02-coding-agent-sandbox-mechanisms.md)：Coding Agent 沙箱机制综述。
5. [`orchestration/01-when-to-use-subagents.md`](orchestration/01-when-to-use-subagents.md)：什么时候该用 subagent。
6. [`workflows/02-ai-assisted-code-review.md`](workflows/02-ai-assisted-code-review.md)：AI 辅助代码 review 的可靠方法。
7. [`workflows/03-plan-documents-for-coding-agents.md`](workflows/03-plan-documents-for-coding-agents.md)：Roadmap、Spec、Execution Plan 和 Status 的文档分层。
8. [`runtime/03-coding-agent-hooks.md`](runtime/03-coding-agent-hooks.md)：Claude Code 与 Codex CLI hooks 的机制、用途和安全边界。
9. [`runtime/04-agent-skills.md`](runtime/04-agent-skills.md)：Agent Skills 如何封装重复工作流、组织知识和验证纪律。
10. [`orchestration/02-multi-agent-research.md`](orchestration/02-multi-agent-research.md)：Multi-Agent 调研。
11. [`workflows/04-vibe-coding-review-and-learning-debt.md`](workflows/04-vibe-coding-review-and-learning-debt.md)：Vibe Coding 的 review 困境和学习债。
