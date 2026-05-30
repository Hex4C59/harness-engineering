# AI Models 文档

这里收纳模型代码能力、reasoning、训练方式、benchmark、模型报告解读和推理基础设施相关文档。

## 目录地图

1. [`capabilities/`](capabilities/README.md)：模型代码能力、reasoning 和 test-time compute。
2. [`evaluation/`](evaluation/README.md)：frontier model benchmark、模型发布报告和评估解读。
3. [`infrastructure/`](infrastructure/README.md)：KV cache、长上下文、prompt caching 和 agent runtime 成本。

## 建议阅读路径

1. [`capabilities/01-how-code-capability-is-trained.md`](capabilities/01-how-code-capability-is-trained.md)：代码能力是怎么训练出来的。
2. [`evaluation/02-understanding-frontier-model-benchmarks.md`](evaluation/02-understanding-frontier-model-benchmarks.md)：如何读懂 Claude、GPT、Gemini 发布时的 benchmark。
3. [`evaluation/03-claude-opus-4-8-report-analysis.md`](evaluation/03-claude-opus-4-8-report-analysis.md)：Claude Opus 4.8 官方报告解读。
4. [`capabilities/04-reasoning-research.md`](capabilities/04-reasoning-research.md)：围绕 reasoning 的论文、模型、benchmark、工程实践、社区争议和历史类比调研。
5. [`infrastructure/05-kv-cache-research.md`](infrastructure/05-kv-cache-research.md)：KV Cache 从 Transformer 推理优化到 agent runtime 基础设施的调研。

相关主题：

- [`../harness-engineering/README.md`](../harness-engineering/README.md)：agent harness 方法论。
- [`../coding-agents/README.md`](../coding-agents/README.md)：coding agent 工具、插件和个人工作流。
