# Codex Notes

这里放 Codex CLI / Codex App 的专属机制和使用经验。

这个目录不再承载通用 coding agent 机制综述。跨工具的 memory、sandbox、hooks、skills 研究放在 [`../research/runtime/`](../research/runtime/README.md)。

适合场景：

- 理解 Codex 上下文压缩、窗口生命周期和实验 Memories。
- 配置或评估 Codex CLI / Codex App 的具体行为。
- 判断 Codex 专属机制和通用 agent runtime 机制之间的关系。

文档：

1. [`context/01-codex-context-compaction-principles.md`](context/01-codex-context-compaction-principles.md)：Codex 上下文压缩原理详解。
2. [`context/02-when-to-start-a-new-codex-session.md`](context/02-when-to-start-a-new-codex-session.md)：什么时候该新开 Codex 窗口，如何在继续、compact 和新会话之间选择。
3. [`memory/01-codex-cli-experimental-memories.md`](memory/01-codex-cli-experimental-memories.md)：Codex CLI 实验 Memories 功能的官方用法、源码观察和个人使用建议。
