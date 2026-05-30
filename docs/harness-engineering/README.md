# Harness Engineering 文档

这里收纳 AI agent 语境里的 **harness engineering** 主题文档。这里的 harness 不是 Harness.io 产品，也不是传统测试里的 test harness，而是围绕模型构建的执行环境、工具边界、状态管理、验证反馈和安全控制系统。

## 目录地图

1. [`foundations/`](foundations/README.md)：harness engineering 的定义、必要性和落地方法。
2. [`applications/`](applications/README.md)：把 harness engineering 映射到具体项目和场景。
3. [`practices/`](practices/README.md)：真实团队、开源项目和论文里的最新实践。
4. [`layers/`](layers/README.md)：prompt、tool use、agent loop、context 和 planning 等 harness 分层。
5. [`references.md`](references.md)：参考资料和延伸阅读。

## 建议阅读路径

1. [`foundations/01-what-is-harness-engineering.md`](foundations/01-what-is-harness-engineering.md)：什么是 harness engineering。
2. [`foundations/02-why-harness-engineering.md`](foundations/02-why-harness-engineering.md)：为什么不能只靠提示词、上下文或更强模型。
3. [`foundations/03-how-to-use-harness-engineering.md`](foundations/03-how-to-use-harness-engineering.md)：怎么在项目中落地 harness。
4. [`applications/01-harness-engineering-for-noclaw.md`](applications/01-harness-engineering-for-noclaw.md)：这套思想如何映射到 noclaw。
5. [`practices/01-latest-harness-engineering-practices.md`](practices/01-latest-harness-engineering-practices.md)：最新 harness engineering 实践经验调研。
6. [`layers/01-prompt-engineering-research.md`](layers/01-prompt-engineering-research.md)：Prompt Engineering 横向调研。
7. [`layers/02-tool-use-research.md`](layers/02-tool-use-research.md)：Tool Use 调研：从函数调用到 agent action layer。
8. [`layers/03-agent-loop-research.md`](layers/03-agent-loop-research.md)：Agent Loop 调研：从 ReAct 循环到生产级 agent runtime。
9. [`layers/04-context-engineering-research.md`](layers/04-context-engineering-research.md)：Context Engineering 调研：agent 系统里的上下文治理工程。
10. [`layers/05-planning-research.md`](layers/05-planning-research.md)：Agent Planning 调研：从模型推理能力到可审计执行控制面。

相关主题：

- [`../coding-agents/README.md`](../coding-agents/README.md)：coding agent 工具、插件和个人工作流。
- [`../ai-models/README.md`](../ai-models/README.md)：模型代码能力、benchmark 和模型报告。
