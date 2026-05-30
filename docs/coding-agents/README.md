# Coding Agents 文档

这里收纳 code CLI、coding agent、agent 插件、个人工作流、sandbox 和 Codex 使用经验相关文档。

这个目录下的文档按用途分成三类：

## Playbooks：写代码时直接用

这些文档更像操作手册和速查卡，适合开新项目、写功能、修 bug、review diff 时直接复制提示词或流程。

正式工作流入口统一放在 [`playbooks/workflows/`](playbooks/workflows/README.md)。旧的根级 playbook 文件不再作为入口；当前目录中也不恢复 `playbooks/01-agent-tdd-project-bootstrap-playbook.md` 和 `playbooks/02-coding-agent-cheatsheet.md`。

1. [`playbooks/README.md`](playbooks/README.md)：可复制到新项目里的 Coding Agent Playbook Toolbox 总入口。
2. [`playbooks/workflows/new-project-bootstrap.md`](playbooks/workflows/new-project-bootstrap.md)：个人 Agent + TDD 新项目 bootstrap 流程。
3. [`playbooks/workflows/existing-project-onboarding.md`](playbooks/workflows/existing-project-onboarding.md)：既有项目接入 agent harness 和 TDD 工作流的渐进式改造流程。
4. [`playbooks/workflows/everyday-development.md`](playbooks/workflows/everyday-development.md)：日常开发、功能、bugfix、review 和上下文恢复入口。
5. [`playbooks/workflows/learning-first-vibe-coding.md`](playbooks/workflows/learning-first-vibe-coding.md)：陌生技术栈下控制 diff、边做边学、独立 review 和避免学习债的工作流。

## Research：跨工具调研、方法论和设计分析

这些文档按主题组织，主要用于阅读、理解和提炼规则，不建议整篇塞进项目根目录。

1. [`research/evaluation/`](research/evaluation/README.md)：Code CLI、模型组合和 agent harness 的评估方法。
2. [`research/runtime/`](research/runtime/README.md)：memory、sandbox、hooks、skills 等 runtime 机制。
3. [`research/orchestration/`](research/orchestration/README.md)：subagent、multi-agent 和任务编排边界。
4. [`research/workflows/`](research/workflows/README.md)：高手工作流、AI review、计划文档和学习优先实践。
5. [`research/case-studies/`](research/case-studies/README.md)：Superpowers、Oh My OpenAgent 等具体项目案例。
6. [`research/strategy/`](research/strategy/README.md)：coding agent 时代的软件工程壁垒和长期判断。

## Codex：Codex 专属机制和使用经验

这些文档用于理解 Codex CLI / Codex App 的具体机制。通用 memory、sandbox、hooks、skills 研究已经放到 `research/runtime/`。

1. [`codex/context/01-codex-context-compaction-principles.md`](codex/context/01-codex-context-compaction-principles.md)：Codex 上下文压缩原理详解。
2. [`codex/context/02-when-to-start-a-new-codex-session.md`](codex/context/02-when-to-start-a-new-codex-session.md)：什么时候该新开 Codex 窗口，如何管理长上下文、compact 和任务交接。
3. [`codex/memory/01-codex-cli-experimental-memories.md`](codex/memory/01-codex-cli-experimental-memories.md)：Codex CLI 实验 Memories 功能的官方用法、源码观察和个人使用建议。

建议阅读路径：

1. 想开新项目：先读 [`playbooks/workflows/new-project-bootstrap.md`](playbooks/workflows/new-project-bootstrap.md)。
2. 想接入既有项目：读 [`playbooks/workflows/existing-project-onboarding.md`](playbooks/workflows/existing-project-onboarding.md)。
3. 日常写代码：打开 [`playbooks/workflows/everyday-development.md`](playbooks/workflows/everyday-development.md) 按场景复制提示词。
4. 想理解高手经验：读 [`research/workflows/01-expert-coding-agent-workflows.md`](research/workflows/01-expert-coding-agent-workflows.md)。
5. 想理解 subagent 使用边界：读 [`research/orchestration/01-when-to-use-subagents.md`](research/orchestration/01-when-to-use-subagents.md)。
6. 想让 AI 生成代码更容易可靠 review：读 [`research/workflows/02-ai-assisted-code-review.md`](research/workflows/02-ai-assisted-code-review.md)。
7. 想设计 roadmap、project-status 和 plans：读 [`research/workflows/03-plan-documents-for-coding-agents.md`](research/workflows/03-plan-documents-for-coding-agents.md)。
8. 想把项目规则、验证、通知和审计沉淀进 agent runtime：读 [`research/runtime/03-coding-agent-hooks.md`](research/runtime/03-coding-agent-hooks.md)。
9. 想理解人人皆可 coding 后真正壁垒在哪里：读 [`research/strategy/01-real-barriers-when-everyone-can-code.md`](research/strategy/01-real-barriers-when-everyone-can-code.md)。
10. 想理解 Agent Skills 如何沉淀重复工作流、组织知识和验证纪律：读 [`research/runtime/04-agent-skills.md`](research/runtime/04-agent-skills.md)。
11. 想系统研究 Multi-Agent：读 [`research/orchestration/02-multi-agent-research.md`](research/orchestration/02-multi-agent-research.md)。
12. 想理解 Vibe Coding 为什么会带来 review 困境和学习债：读 [`research/workflows/04-vibe-coding-review-and-learning-debt.md`](research/workflows/04-vibe-coding-review-and-learning-debt.md)。
13. 想把 Vibe Coding 的判断落实成可复制 prompt：读 [`playbooks/workflows/learning-first-vibe-coding.md`](playbooks/workflows/learning-first-vibe-coding.md)。
14. 想理解 Codex 机制：从 [`codex/context/01-codex-context-compaction-principles.md`](codex/context/01-codex-context-compaction-principles.md) 开始。
15. 想研究 Codex 实验 Memories 功能：读 [`codex/memory/01-codex-cli-experimental-memories.md`](codex/memory/01-codex-cli-experimental-memories.md)。

相关主题：

- [`../harness-engineering/README.md`](../harness-engineering/README.md)：agent harness 方法论。
- [`../ai-models/README.md`](../ai-models/README.md)：模型代码能力、benchmark 和模型报告。
